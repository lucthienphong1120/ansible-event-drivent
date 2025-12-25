## Demo: Use EDA Server

Here is a demo of configuring a webhook on the EDA Server side, and triggering a Job Template on AWX by posting payload that contains a specific `message` to the webhook.

In this demo, following example Rulebook is used. Review the Rulebook.

- **Webhook as a source**: [demo_webhook.yaml](demo_webhook.yaml)
  - The webhook that listening on `0.0.0.0:5000` is defined as a source of the Ruleset.
  - This Ruleset has a rule that if the payload contains `message` field with the body `Hello EDA`, trigger `Demo Job Template` in `Default` organization on AWX.

In addition to the webhook demo, a quick demo to use MQTT as a source is also provided.

- **MQTT as a source**: [demo_mqtt.yaml](demo_mqtt.yaml)
  - As a source of the Ruleset, subscribing MQTT topic on the MQTT broker is defined. Actual connection information for MQTT can be defined by Rulebook Variables.
  - This Ruleset has a rule that if the received data contains `message` field with the body `Hello EDA`, trigger `Demo Job Template` in `Default` organization on AWX.

### Configure EDA Server

In order to the webhook to be ready to receive messages, the following tasks need to be done.

- Issue new token for AWX and add it on EDA Server
- Add Decision Environment on EDA Server
- Add Project on EDA Server
- Enable Rulebook Activation
- Deploy Ingress resource for the webhook

#### Issue new token for AWX and add it on EDA Server

EDA Server uses a token to access AWX. This token has to be issued by AWX and registered on EDA Server.

To issue new token by AWX, in the Web UI for AWX, open `User Details` page (accessible by user icon at the upper right corner), follow to the `Tokens` tab, and then click `Add` button. Specify `Write` as `Scope` and click `Save`, then keep the issued token in the safe place.

Alternatively we can issue new token by CLI as follows:

```bash
$ kubectl -n awx exec deployment/awx-task -- awx-manage create_oauth2_token --user=admin
4sIZrWXi**************8xChmahb
```

To register the token on EDA Server, in the Web UI for EDA Server, open `User details` page (accessible by user icon at the upper right corner), follow to the `Controller Tokens` tab, and then click `Create controller token` button.

Fill the form as follows, then click `Create controller token` button on the bottom of the page:

| Key | Value |
| - | - |
| Name | `awx.example.com` |
| Token | `<YOUR_TOKEN>` |

#### Add Decision Environment on EDA Server

Decision Environment (DE) is an environment for running Ansible Rulebook (`ansible-rulebook`) by the EDA Server, like Execution Environment (EE) for running Ansible Runner (`ansible-runner`) by the AWX.

There is no default DE on EDA Server, so we have to register new one.

Open `Decision Environments` on Web UI for EDA Server, then click `Create decision environment` button.

Fill the form as follows, then click `Create decision environment` button on the bottom of the page:

| Key | Value |
| - | - |
| Name | `DE Latest` |
| Image | `quay.io/ansible/ansible-rulebook:latest` |

#### Add Project on EDA Server

To run Ansible Rulebook by EDA Server, the repository on SCM `must contains rulebooks folder` have to be registered as Project on EDA Server.

This repository contains some example Rulebooks under [rulebooks](./) directory, so we can register this repository as Project.

Open `Projects` on Web UI for EDA Server, then click `Create project` button.

Fill the form as follows, then click `Create project` button on the bottom of the page:

| Key | Value |
| - | - |
| Name | `Demo Project` |
| SCM URL | `https://github.com/lucthienphong1120/ansible-event-drivent.git` |

Refresh the page and wait for the `Status` for the project to be `Completed`.

#### Enable Rulebook Activation

To run Ansible Rulebook by EDA Server, define Rulebook Activation and enable it.

Open `Rulebook Activations` on Web UI for EDA Server, then click `Create rulebook activation` button.

Fill the form as follows, then click `Create rulebook activation` button on the bottom of the page:

| Key | Value |
| - | - |
| Name | `Trigger Demo Job Template by Webhook` |
| Project | `Demo Project` |
| Rulebook | `demo_webhook.yaml` |
| Decision environment | `Minimal DE` |
| Controller token | `awx.example.com` |

Refresh the page and wait for the `Activation status` for the Rulebook to be `Running`.

#### Identify activation job

When you enable Rulebook Activation, a Job called an activation job is launched.

```bash
$ kubectl -n eda get job
NAME                 COMPLETIONS   DURATION   AGE
activation-job-1-1   0/1           7m3s       7m3s
```

The name of the activation job will be changed with each enabling Rulebook Activation, so it is important to know this name for subsequent tasks.

As an example above, the name of the activation job contains two IDs, such as `activation-job-<Activation ID>-<Instance ID>`. Each of these IDs can be identified as follows.

- **Activation ID**
  - This is an ID for your Rulebook Activation.
  - This is uniquely assigned to each Rulebook Activation and does not change.
  - On Web UI, you can gather this ID by `ID` column on `Rulebook Activations` page, or `Activation ID` on the `Details` tab for your Rulebook Activation.
- **Instance ID**
  - This is an ID of each instance of the Ansible Rulebook.
  - It is newly assigned each time a Rulebook Activation is enabled. It means that the ID changes when Rulebook Activation is disabled and re-enabled.
  - On Web UI, you can gather this ID by `Name` on the `History` tab for your Rulebook Activation. The number at the beginning of the `Name` column of the line that in `Running` state is the ID.

Once the activation job name has been identified by collecting those two IDs, the Pod created by this job can also be identified. This is the Pod where `ansible-rulebook` is running on.

```bash
$ JOB_NAME=activation-job-1-1
$ kubectl -n eda get pod -l job-name=${JOB_NAME}
NAME                       READY   STATUS    RESTARTS   AGE
activation-job-1-1-ctz24   1/1     Running   0          7m16s
```

The new Service is also created by EDA Server. This Service provides the endpoint for the webhook by routing traffic to the above Pod.

```bash
$ kubectl -n eda get service -l job-name=${JOB_NAME}
NAME                      TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
activation-job-1-1-5000   ClusterIP   10.43.82.223   <none>        5000/TCP   7m40s
```

#### Deploy Ingress resource for the webhook

To make the webhook externally accessible, we have to expose the Service that created by EDA Server.

To achieve this, in this example, we create new Ingress.

Modify `hosts`, `host`, and `name` under `service` in `rulebooks/webhook/ingress.yaml`. Here, the same hostname as the EDA Server are specified so that the endpoint for webhook can be accessed under the same URL as the EDA Server. Note that the `name` of the `service` has to be the name of the Service that created by EDA Server, as reviewed above.

```yaml
...
spec:
  tls:
    - hosts:
        - eda.example.com                       👈👈👈
      secretName: eda-secret-tls
  rules:
    - host: eda.example.com                     👈👈👈
      http:
        paths:
          - path: /webhooks/demo
            pathType: ImplementationSpecific
            backend:
              service:
                name: activation-job-1-1-5000   👈👈👈
                port:
                  number: 5000
```

By applying this file, your webhook can be accessed on the URL `https://eda.example.com/webhooks/demo`.

```bash
$ kubectl apply -f rulebooks/webhook/ingress.yaml
...

$ kubectl -n eda get ingress
NAME                  CLASS     HOSTS             ADDRESS         PORTS     AGE
eda-ingress           traefik   eda.example.com   192.168.0.221   80, 443   4h45m
eda-ingress-webhook   traefik   eda.example.com   192.168.0.221   80, 443   1s   👈👈👈
```

### Trigger Rule using Webhook

In [the Rulebook (`demo_webhook.yaml`) that used in this demo](demo_webhook.yaml), the Job Template (`Demo Job Template`) is invoked on AWX on the condition that `message` in the payload that posted to the webhook is `Hello EDA`.

Post that payload to the webhook, and review the Job Template is triggered.

```bash
$ curl -k \
  -H "Content-Type: application/json" \
  -d '{"message": "Hello EDA"}' \
  https://eda.example.com/webhooks/demo
```

Review `Rule Audit` page on the Web UI for EDA Server, and `Jobs` page under `Views` on the Web UI for AWX.

### Appendix: Use MQTT as a source

For a more authentic example, here is an example of using MQTT as a source.

To use MQTT, a MQTT broker is required. If you don't have one, you can use `kubectl apply -k rulebooks/mqtt/broker` to deploy a minimal MQTT broker on K3s that listens on `31883` or use a public broker such as [test.mosquitto.org](https://test.mosquitto.org/). Also, this demonstration is prepared assuming that neither authentication nor encryption is used.

Define the Decision Environment with the following information, just as you configured for the webhook.

| Key | Value |
| - | - |
| Name | `Minimal DE with MQTT` |
| Image | `docker.io/kurokobo/ansible-rulebook:v1.0.6-mqtt` |

Note that the image specified above is based on `quay.io/ansible/ansible-rulebook:stable-1.0` and includes [`kubealex.eda`](https://galaxy.ansible.com/ui/repo/published/kubealex/eda/) collection that includes `kubealex.eda.mqtt` plugin. The Dockerfile for this image is available under [mqtt/de directory](./mqtt/de).

Then define Rulebook Activation as follows. Note that you should modify actual values for `Variables` to suit your environment:

| Key | Value |
| - | - |
| Name | `Trigger Demo Job Template by MQTT` |
| Project | `Demo Project` |
| Rulebook | `demo_mqtt.yaml` |
| Decision environment | `Minimal DE with MQTT` |
| Controller token | `awx.example.com` |
| Variables | `mqtt_host: mqtt.example.com`<br>`mqtt_port: 31883`<br>`mqtt_topic: demo` |

Activate the Rulebook, and publish specific message that matches the condition in the Rule to the topic you've defined.

```bash
docker run -it --rm efrecon/mqtt-client pub \
        -h mqtt.example.com \
        -p 31883 \
        -t demo \
        -m '{"message": "Hello EDA"}'
```

Ensure your Job Template on AWX has been triggered.
