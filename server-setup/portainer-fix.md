Portainer fix mixed with existing config

https://github.com/orgs/portainer/discussions/12926#discussioncomment-15049303
```
{
  "min-api-version": "1.24",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "features": {
    "containerd-snapshotter": false
  },
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "args": [],
      "path": "nvidia-container-runtime"
    }
  },
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/80",
  "experimental": true,
  "ip6tables": true
}
```

Old Config
```
{
  "default-runtime": "nvidia",
  "runtimes": {
    "nvidia": {
      "args": [],
      "path": "nvidia-container-runtime"
    }
  },
  "ipv6": true,
  "fixed-cidr-v6": "fd00::/80",
  "experimental": true,
  "ip6tables": true
}
```
