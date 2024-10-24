% ramalama-serve 1

## NAME
ramalama\-serve - serve REST API on specified AI Model

## SYNOPSIS
**ramalama serve** [*options*] _model_

## DESCRIPTION
Serve specified AI Model as a chat bot. RamaLama pulls specified AI Model from
registry if it does not exist in local storage.

## REST API ENDPOINTS
Under the hood, `ramalama-serve` uses the `LLaMA.cpp` HTTP server by default.

For REST API endpoint documentation, see: [https://github.com/ggerganov/llama.cpp/blob/master/examples/server/README.md#api-endpoints](https://github.com/ggerganov/llama.cpp/blob/master/examples/server/README.md#api-endpoints)

## OPTIONS

#### **--detach**, **-d**
Run the container in the background and print the new container ID.
The default is TRUE. The --nocontainer option forces this option to False.

Use the `ramalama stop` command to stop the container running the served ramalama Model.

#### **--generate**=type
Generate specified configuration format for running the AI Model as a service

| Key       | Description                                                      |
| --------- | ---------------------------------------------------------------- |
|  quadlet  | Podman supported container definition for running AI Model under systemd |
|  kube     | Kubernetes YAML definition for running the AI Model as a service |

#### **--help**, **-h**
show this help message and exit

#### **--name**, **-n**
Name of the container to run the Model in.

#### **--port**, **-p**
port for AI Model server to listen on

## EXAMPLES
### Run two AI Models at the same time. Notice both are running within Podman Containers.
```

$ ramalama serve -p 8080 --name mymodel ollama://tiny-llm:latest
09b0e0d26ed28a8418fb5cd0da641376a08c435063317e89cf8f5336baf35cfa

$ ramalama serve -n example --port 8081 oci://quay.io/mmortari/gguf-py-example/v1/example.gguf
3f64927f11a5da5ded7048b226fbe1362ee399021f5e8058c73949a677b6ac9c

$ podman ps
CONTAINER ID  IMAGE                             COMMAND               CREATED         STATUS         PORTS                   NAMES
09b0e0d26ed2  quay.io/ramalama/ramalama:latest  /usr/bin/ramalama...  32 seconds ago  Up 32 seconds  0.0.0.0:8081->8081/tcp  ramalama_sTLNkijNNP
3f64927f11a5  quay.io/ramalama/ramalama:latest  /usr/bin/ramalama...  17 seconds ago  Up 17 seconds  0.0.0.0:8082->8082/tcp  ramalama_YMPQvJxN97
```

### Generate a quadlet for running the AI Model service
```
$ ramalama serve --name MyGraniteServer --generate=quadlet granite > $HOME/.config/containers/systemd/MyGraniteServer.container
$ cat $HOME/.config/containers/systemd/MyGraniteServer.container

[Unit]
Description=RamaLama granite AI Model Service
After=local-fs.target

[Container]
AddDevice=-/dev/dri
AddDevice=-/dev/kfd
Exec=llama-server --port 8080 -m /home/dwalsh/.local/share/ramalama/models/huggingface/instructlab/granite-7b-lab-GGUF/granite-7b-lab-Q4_K_M.gguf
Image=quay.io/ramalama/ramalama:latest
Volume=/home/dwalsh/.local/share/ramalama/models/huggingface/instructlab/granite-7b-lab-GGUF/granite-7b-lab-Q4_K_M.gguf:/home/dwalsh/.local/share/ramalama/models/huggingface/instructlab/granite-7b-lab-GGUF/granite-7b-lab-Q4_K_M.gguf:ro,z
ContainerName=MyGraniteServer
PublishPort=8080

[Install]
# Start by default on boot
WantedBy=multi-user.target default.target
$ systemctl --user daemon-reload
$ systemctl start --user MyGraniteServer
$ systemctl status --user MyGraniteServer
● MyGraniteServer.service - RamaLama granite AI Model Service
     Loaded: loaded (/home/dwalsh/.config/containers/systemd/MyGraniteServer.container; generated)
    Drop-In: /usr/lib/systemd/user/service.d
	     └─10-timeout-abort.conf
     Active: active (running) since Fri 2024-09-27 06:54:17 EDT; 3min 3s ago
   Main PID: 3706287 (conmon)
      Tasks: 20 (limit: 76808)
     Memory: 1.0G (peak: 1.0G)
...
$ podman ps
CONTAINER ID  IMAGE                             COMMAND               CREATED        STATUS        PORTS                    NAMES
7bb35b97a0fe  quay.io/ramalama/ramalama:latest  llama-server --po...  3 minutes ago  Up 3 minutes  0.0.0.0:43869->8080/tcp  MyGraniteServer
```

### Generate a kubernetes YAML file named tini
```
$ ramalama serve --name tini --generate kube tiny
# Save the output of this file and use kubectl create -f to import
# it into Kubernetes.
#
# Created with ramalama-0.0.17
apiVersion: v1
kind: Deployment
metadata:
  labels:
    app: tini
  name: tini
spec:
  containers:
  - name: tini
    image: quay.io/ramalama/ramalama:latest
    command: ["llama-server"]
    args: ['--port', '8080', '-m', '/run/model']
    ports:
    - containerPort: 8080
    volumeMounts:
    - mountPath: /run/model
      name: model
    - mountPath: /dev/dri
      name: dri
  volumes:
  - name model
    hostPath:
      path: /home/dwalsh/.local/share/ramalama/models/ollama/tinyllama:latest"
  - name dri
    hostPath:
      path: /dev/dri
```

## SEE ALSO
**[ramalama(1)](ramalama.1.md)**, **[ramalama-stop(1)](ramalama-stop.1.md)**, **quadlet(1)**, **systemctl(1)**, **podman-ps(1)**

## HISTORY
Aug 2024, Originally compiled by Dan Walsh <dwalsh@redhat.com>
