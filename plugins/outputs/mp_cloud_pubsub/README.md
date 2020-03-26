# Multiplay - Google Cloud PubSub Output Plugin

The GCP PubSub plugin publishes metrics to a [Google Cloud PubSub][pubsub] topic
as one of the supported [output data formats][].

It is based upon the standard Telegraf [cloud_pubsub] plugin but uses Vault
as a backend for fetching the credentials file secret.

### Vault Setup
Connection to Pub/Sub is required to use a service account file, the contents
of which much exist in your Vault server at the following path:

```
vault kv put secret/pubsub-credentials value=path/to/file.json
```

To access this secret you first need to create a policy with read access. Create
a file called `pol.hcl` with the following contents:

```
path "secret/data/*" {
  capabilities = [ "read" ]
}
```

Apply this policy using the Vault CLI:
```bash
$ vault policy write my-policy pol.hcl
```

In order for the plugin to access the secret we need to use an [AppRole]. Ensure
this auth method is enabled on your Vault server:
```bash
$ vault auth enable approle
```

The AppRole needs to have access to the policy you just created. To create an app
role run this command:
```bash
$ vault write -f auth/approle/role/my-role token_policies="my-policy"
```

Next we need to get credentials for the app role that we can store in our
application. First you can get the role ID like so:
```bash
$ vault read auth/approle/role/my-role/role-id
```

Add the role ID to your Telegraf config for this plugin. Finally you need to
generate a secret ID:
```bash
$ vault write -f auth/approle/role/my-role/secret-id
```
Add the secret ID to your Telegraf config for this plugin.

### Configuration

This section contains the default TOML to configure the plugin.  You can
generate it using `telegraf --usage cloud_pubsub`.

```toml
[[outputs.cloud_pubsub]]
  ## Required. Name of Google Cloud Platform (GCP) Project that owns
  ## the given PubSub topic.
  project = "my-project"

  ## Required. Name of PubSub topic to publish metrics to.
  topic = "my-topic"

  ## Required. Data format to consume.
  ## Each data format has its own unique set of configuration options.
  ## Read more about them here:
  ## https://github.com/influxdata/telegraf/blob/master/docs/DATA_FORMATS_INPUT.md
  data_format = "influx"

  ## Required. Full URL to the Vault server.
  vault_address = "http://127.0.0.1:8200"

  ## Required. Role ID of an AppRole that has appropriate access to the GCP
  ## service account secret in Vault.
  vault_role_id = "my-role-id"

  ## Required. Secret ID of an AppRole that has appropriate access to the GCP
  ## service account secret in Vault.
  vault_secret_id = "my-secret-id"

  ## Optional. If true, will send all metrics per write in one PubSub message.
  # send_batched = true

  ## The following publish_* parameters specifically configures batching
  ## requests made to the GCP Cloud PubSub API via the PubSub Golang library. Read
  ## more here: https://godoc.org/cloud.google.com/go/pubsub#PublishSettings

  ## Optional. Send a request to PubSub (i.e. actually publish a batch)
  ## when it has this many PubSub messages. If send_batched is true,
  ## this is ignored and treated as if it were 1.
  # publish_count_threshold = 1000

  ## Optional. Send a request to PubSub (i.e. actually publish a batch)
  ## when it has this many PubSub messages. If send_batched is true,
  ## this is ignored and treated as if it were 1
  # publish_byte_threshold = 1000000

  ## Optional. Specifically configures requests made to the PubSub API.
  # publish_num_go_routines = 2

  ## Optional. Specifies a timeout for requests to the PubSub API.
  # publish_timeout = "30s"

  ## Optional. If true, published PubSub message data will be base64-encoded.
  # base64_data = false

  ## Optional. PubSub attributes to add to metrics.
  # [[inputs.pubsub.attributes]]
  #   my_attr = "tag_value"
```

[pubsub]: https://cloud.google.com/pubsub
[output data formats]: /docs/DATA_FORMATS_OUTPUT.md
[cloud_pubsub]: /plugins/outputs/cloud_pubsub
[AppRole]: https://www.vaultproject.io/docs/auth/approle/
