# SiteEncrypt

[![hex.pm](https://img.shields.io/hexpm/v/parent.svg?style=flat-square)](https://hex.pm/packages/site_encrypt)
[![hexdocs.pm](https://img.shields.io/badge/docs-latest-green.svg?style=flat-square)](https://hexdocs.pm/site_encrypt/)
![Build Status](https://github.com/sasa1977/site_encrypt/workflows/site_encrypt/badge.svg)

This project aims to provide integrated certification via [Let's encrypt](https://letsencrypt.org/) for sites implemented in Elixir.

Integrated certification means that you don't need to run any other OS process in background. Start your site for the first time, and the system will obtain the certificate, and periodically renew it before it expires.

The target projects are small-to-medium Elixir based sites which don't sit behind reverse proxies such as nginx.

## Status

- The library is tested in a [simple production](https://www.theerlangelist.com), where it has been constantly running since mid 2018.
- Native Elixir client is very new, and not considered stable. If you prefer reliable behaviour, use the Certbot client. This will require installing [Certbot](https://certbot.eff.org/) >= 0.31
- The API is not stable. Expect breaking changes in the future.

## Quick start

A basic demo Phoenix project is available [here](https://github.com/sasa1977/site_encrypt/tree/master/demos/phoenix).

1. Add the dependency to `mix.exs`:

    ```elixir
    defmodule PhoenixDemo.Mixfile do
      # ...

      defp deps do
        [
          # ...
          {:site_encrypt, "~> 0.5"}
        ]
      end
    end
    ```

    Don't forget to invoke `mix.deps` after that.

1. Expand your endpoint

    ```elixir
    defmodule PhoenixDemo.Endpoint do
      # ...

      # add this after `use Phoenix.Endpoint`
      use SiteEncrypt.Phoenix

      # ...

      @impl SiteEncrypt
      def certification do
        SiteEncrypt.configure(
          # Note that native client is very immature. If you want a more stable behaviour, you can
          # provide `:certbot` instead. Note that in this case certbot needs to be installed on the
          # host machine.
          client: :native,

          domains: ["mysite.com", "www.mysite.com"],
          emails: ["contact@abc.org", "another_contact@abc.org"],

          # By default the certs will be stored in tmp/site_encrypt_db, which is convenient for
          # local development. Make sure that tmp folder is gitignored.
          #
          # Set OS env var SITE_ENCRYPT_DB on staging/production hosts to some absolute path
          # outside of the deployment folder. Otherwise, the deploy may delete the db_folder,
          # which will effectively remove the generated key and certificate files.
          db_folder:
            System.get_env("SITE_ENCRYPT_DB", Path.join("tmp", "site_encrypt_db")),

          # set OS env var CERT_MODE to "staging" or "production" on staging/production hosts
          directory_url:
            case System.get_env("CERT_MODE", "local") do
              "local" -> {:internal, port: 4002}
              "staging" -> "https://acme-staging-v02.api.letsencrypt.org/directory"
              "production" -> "https://acme-v02.api.letsencrypt.org/directory"
            end
        )
      end

      # ...
    end
    ```

1. Configure https:

    ```elixir
    defmodule PhoenixDemo.Endpoint do
      # ...

      @impl Phoenix.Endpoint
      def init(_key, config) do
        # this will merge key, cert, and chain into `:https` configuration from config.exs
        {:ok, SiteEncrypt.Phoenix.configure_https(config)}

        # to completely configure https from `init/2`, invoke:
        #   SiteEncrypt.Phoenix.configure_https(config, port: 4001, ...)
      end

      # ...
    end
    ```

1. Start the endpoint via `SiteEncrypt`:

    ```elixir
    defmodule PhoenixDemo.Application do
      use Application

      def start(_type, _args) do
        children = [{SiteEncrypt.Phoenix, PhoenixDemo.Endpoint}]
        opts = [strategy: :one_for_one, name: PhoenixDemo.Supervisor]
        Supervisor.start_link(children, opts)
      end

      # ...
    end
    ```

1. Optionally add a certification test

    ```elixir
    defmodule PhoenixDemo.Endpoint.CertificationTest do
      use ExUnit.Case, async: false
      import SiteEncrypt.Phoenix.Test

      test "certification" do
        clean_restart(PhoenixDemo.Endpoint)
        cert = get_cert(PhoenixDemo.Endpoint)
        assert cert.domains == ~w/mysite.com www.mysite.com/
      end
    end
    ```

And that's it! At this point you can start the system:

```text
$ iex -S mix phx.server

[info]  Generating a temporary self-signed certificate. This certificate will be used until a proper certificate is issued by the CA server.
[info]  Running PhoenixDemo.Endpoint with cowboy 2.7.0 at 0.0.0.0:4000 (http)
[info]  Running PhoenixDemo.Endpoint with cowboy 2.7.0 at 0.0.0.0:4001 (https)
[info]  Running local ACME server at port 4002
[info]  Ordering a new certificate for domain mysite.com
[info]  New certificate for domain mysite.com obtained
[info]  Certificate successfully obtained!
```

And visit your certified site at https://localhost:4001

## Testing in production

In general, the configuration above should work out of the box in production, as long as the domain is correctly setup, and ports properly forwarded, so the HTTP site is externally available at port 80.

If you want a more manual first deploy test, here's how you can do it:

1. Explicitly set `mode: :manual` in `certification/0`. This means that the site won't automatically certify itself. However, during the first boot it will generate a self-signed certificate.

2. Deploy the site and verify that it's externally reachable via HTTP on port 80.

3. Start a remote `iex` shell session to the running system.

4. Perform a trial certification through the staging Let's Encrypt CA:

    ```elixir
    iex> SiteEncrypt.dry_certify(
           MySystemWeb.Endpoint,
           directory_url: "https://acme-staging-v02.api.letsencrypt.org/directory"
         )
    ```

  Keep in mind that this can be only invoked in the remote `iex` shell session inside the running system.

  If the certification succeeds, the function will return the key and the certificate. These files won't be stored on disk, and they won't be used by the endpoint.

5. If the trial certification succeeded, you can proceed to start the real certification as follows:

    ```elixir
    iex> SiteEncrypt.force_certify(MySystemWeb.Endpoint)
    ```

  Unlike the trial certification, this function will go to the CA as configured by the `certification/0` callback in the endpoint. The key and the certificate files will be stored on the disk, and the site will immediately used them. Therefore, if this function succeeds, you can visit your site via HTTPS.

6. If all went well, remove the `:mode` setting from the `certification/0` callback and redeploy your system.

__Note__: be careful not to invoke these functions too frequently, because you might trip some rate limit on Let's Encrypt. See [here](https://letsencrypt.org/docs/rate-limits/) for more details.

## Clustering

SiteEncrypt currently provides minimal support for nodes running in a cluster, so this must largely be handled by your system. There are two
callbacks provided that can be used to provide some support: 

`registered_challenge/3` is called after a node contacts the ACME server and generates an authorization key pair
`got_challenge/2` is called whenever an ACME request endpoint is reached, before the node responds to the challenge


These can be used by your application to ensure that the challenge token is available to all nodes regardless of which node initiates
the request or which node receives the callback:

for example, you could add in your endpoint
```elixir
  @impl SiteEncrypt
  def registered_challenge(_id, challenge_token, key_thumbprint) do
    saved_acme_challenge(challenge_token)
    |> File.write(key_thumbprint)
  end

  @impl SiteEncrypt
  def got_challenge(id, challenge_token) do
    saved_acme_challenge(challenge_token)
    |> File.read()
    |> case do
      {:ok, key_thumbprint} ->
        SiteEncrypt.Registry.register_challenge(id, challenge_token, key_thumbprint, false)

      _ ->
        :ok
    end

    :ok
  end

  defp saved_acme_challenge(challenge_token) do
    # FIXME: generate a safe path, being careful to avoid path traversal vulnerabilities
  end
```
to save the challenge in a shared folder. If your cluster is large or will be restarted without keeping the saved certificate frequently,
you may also need to change the SiteEncrypt.config call to ensure you do not breach the ACME server's rate limits.


## License

[MIT](./LICENSE)
