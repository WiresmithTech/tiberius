# Tiberius
[![crates.io](https://meritbadge.herokuapp.com/tiberius)](https://crates.io/crates/tiberius)
[![docs.rs](https://docs.rs/tiberius/badge.svg)](https://docs.rs/tiberius)
[![Cargo tests](https://github.com/prisma/tiberius/actions/workflows/test.yml/badge.svg)](https://github.com/prisma/tiberius/actions/workflows/test.yml)
[![Chat](https://img.shields.io/discord/664092374359605268)](https://discord.gg/xX4xp9x)

A native Microsoft SQL Server (TDS) client for Rust.

### Goals

- A perfect implementation of the TDS protocol.
- Asynchronous network IO.
- Independent of the protocol.
- Support for latest versions of Linux, Windows and macOS.

### Non-goals

- Connection pooling (use [bb8](https://crates.io/crates/bb8), [mobc](https://crates.io/crates/mobc), [deadpool](https://crates.io/crates/deadpool) or any of the other asynchronous connection pools)
- Query building
- Object-relational mapping

### Supported SQL Server versions

| Version | Support level | Notes                               |
|---------|---------------|-------------------------------------|
|    2019 | Tested on CI  |                                     |
|    2017 | Tested on CI  |                                     |
|    2016 | Should work   |                                     |
|    2014 | Should work   |                                     |
|    2012 | Should work   |                                     |
|    2008 | Should work   |                                     |
|    2005 | Should work   | With feature flag `tds73` disabled. |

The system should work with the Docker and Azure versions of SQL Server without
trouble. If using the Windows installer of SQL Server, either TCP or named pipes protocols
should be enabled in the server settings for Tiberius to be able to connect.

### Feature flags

| Flag                     | Description                                                                           | Default    |
|--------------------------|---------------------------------------------------------------------------------------|------------|
| `tds73`                  | Support for new date and time types in TDS version 7.3. Disable if using version 7.2. | `enabled`  |
| `chrono`                 | Read and write date and time values using `chrono`'s types.                           | `disabled` |
| `rust_decimal`           | Read and write `numeric`/`decimal` values using `rust_decimal`'s `Decimal`.           | `disabled` |
| `bigdecimal`             | Read and write `numeric`/`decimal` values using `bigdecimal`'s `BigDecimal`.          | `disabled` |
| `sql-browser-async-std`  | SQL Browser implementation for the `TcpStream` of async-std.                          | `disabled` |
| `sql-browser-tokio`      | SQL Browser implementation for the `TcpStream` of Tokio.                              | `disabled` |
| `sql-browser-smol`      | SQL Browser implementation for the `TcpStream` of smol.                              | `disabled` |
| `integrated-auth-gssapi` | Support for using Integrated Auth via GSSAPI                                          | `disabled` |
| `vendored-openssl`       | On Linux and macOS platforms links statically against a vendored version of OpenSSL   | `disabled` |

### Supported protocols

Tiberius does not rely on any protocol when connecting to an SQL Server instance. Instead the `Client` takes a socket that implements the `AsyncRead` and `AsyncWrite` traits from the [futures-rs](https://crates.io/crates/futures) crate.

Currently there are good async implementations for TCP in the [async-std](https://crates.io/crates/async-std), [Tokio](https://crates.io/crates/tokio) and [Smol](https://crates.io/crates/smol) projects. To be able to use them together with Tiberius on Windows platforms with SQL Server, the TCP should be enabled in the [server settings](https://technet.microsoft.com/en-us/library/hh231672(v=sql.110).aspx) (disabled by default). In the offficial [Docker image](https://hub.docker.com/_/microsoft-mssql-server) TCP is is enabled by default.

Named pipes should work by using the [NamedPipeClient](https://docs.rs/tokio/1.9.0/tokio/net/windows/named_pipe/struct.NamedPipeClient.html) from the latest Tokio versions.

The shared memory protocol is not documented and seems there are no Rust crates implementing it.

### Encryption (TLS/SSL)

Tiberius has three encryption settings:

| Encryption level | Description                                      |
|------------------|--------------------------------------------------|
| `Required`       | All traffic is encrypted.                        |
| `Off`            | Only the login procedure is encrypted. (default) |
| `NotSupported`   | None of the traffic is encrypted.                |

Some SQL Server databases, such as the public Docker image use a TLS certificate not accepted by Apple's Secure Transport. Therefore on macOS systems we use OpenSSL instead of Secure Transport, meaning by default Tiberius requires a working OpenSSL installation. By using a feature flag `vendored-openssl` the compilation links statically to a vendored version of OpenSSL, allowing encrypted connections from macOS.

Please be aware of the security implications if deciding to use vendoring.

### Integrated Authentication (TrustedConnection) on \*nix

With the `integrated-auth-gssapi` feature enabled, the crate requires the GSSAPI/Kerberos libraries/headers installed:
  * [CentOS](https://pkgs.org/download/krb5-devel)
  * [Arch](https://www.archlinux.org/packages/core/x86_64/krb5/)
  * [Debian](https://tracker.debian.org/pkg/krb5) (you need the -dev packages to build)
  * [Ubuntu](https://packages.ubuntu.com/bionic-updates/libkrb5-dev)
  * NixOS: Run `nix-shell shell.nix` on the repository root.
  * Mac: as of version `0.4.2` the [libgssapi](https://crates.io/crates/libgssapi) crate used for this feature now uses Apple's [GSS Framework](https://developer.apple.com/documentation/gss?language=objc) which ships with MacOS 10.14+.

Additionally, your runtime system will need to be trusted by and configured for the Active Directory domain your SQL Server is part of. In particular, you'll need to be able to get a valid TGT for your identity, via `kinit` or a keytab. This setup varies by environment and OS, but your friendly network/system administrator should be able to help figure out the specifics.

## Redirects

With certain Azure firewall settings, a login might return `Error::Routing { host, port }`. This means the user must create a new `TcpStream` to the given address, and connect again.

A simple connection procedure would then be:

```rust
use tiberius::{Client, Config, AuthMethod, error::Error};
use tokio_util::compat::TokioAsyncWriteCompatExt;
use tokio::net::TcpStream;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let mut config = Config::new();

    config.host("0.0.0.0");
    config.port(1433);
    config.authentication(AuthMethod::sql_server("SA", "<Mys3cureP4ssW0rD>"));

    let tcp = TcpStream::connect(config.get_addr()).await?;
    tcp.set_nodelay(true)?;

    let client = match Client::connect(config, tcp.compat_write()).await {
        // Connection successful.
        Ok(client) => client,
        // The server wants us to redirect to a different address
        Err(Error::Routing { host, port }) => {
            let mut config = Config::new();

            config.host(&host);
            config.port(port);
            config.authentication(AuthMethod::sql_server("SA", "<Mys3cureP4ssW0rD>"));

            let tcp = TcpStream::connect(config.get_addr()).await?;
            tcp.set_nodelay(true)?;

            // we should not have more than one redirect, so we'll short-circuit here.
            Client::connect(config, tcp.compat_write()).await?
        }
        Err(e) => Err(e)?,
    };

    Ok(())
}
```

## Security

If you have a security issue to report, please contact us at [security@prisma.io](mailto:security@prisma.io?subject=[GitHub]%20Prisma%202%20Security%20Report%20Tiberius)
