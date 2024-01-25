<!--
title: Cross-VLAN Home Assistant inside K8S
description: Running Home Assistant across VLANs in Kubernetes without using the host network
image: https://library.wamphlett.net/photos/blog/homelab/home-assistant-k8s.png
slug: home-assistant-in-kubernetes
published: 2024-01-25
-->
# Cross-VLAN Home Assistant inside K8S
I have been running Home Assistant in [Microk8s](https://microk8s.io/) for several years now without any issues. Simply just running the [Home Assistant core container](https://github.com/home-assistant/core/pkgs/container/home-assistant) with a persistent storage claim.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: home-assistant
spec:
  replicas: 1
  selector:
    matchLabels:
      app: home-assistant
  template:
    metadata:
      labels:
        app: home-assistant
    spec:
      hostNetwork: true
      containers:
        - name: home-assistant
          image: ghcr.io/home-assistant/home-assistant:2023.1
          ports:
            - containerPort: 8123
          volumeMounts:
            - name: home-assistant-storage
              mountPath: /config
          env:
            - name: TZ
              value: Europe/London

      volumes:
        - name: home-assistant-storage
          persistentVolumeClaim:
            claimName: home-assistant-storage-claim
```

And its really that easy. Deploy it. Serve it. Enjoy.

## So why change it?
Recently I gave my home lab a big refresh. I moved away from Microk8s in favour of [K3S](https://k3s.io/) and became a lot stricter with my network security. Although I have always segregated my IoT devices onto their own network, there has always been a few exceptions to the rule - mostly thanks to network discovery. My Home Assistant also ran on the host network too, this is the most suggested thing on the internet for people who end up in this position, and sure, its super convenient, one line and all your discoverability issues go away.

```yaml
hostNetwork: true
```

The problem with this approach is, it increases your attack surface. Okay, you might not think about that immediately and it might not even matter if you're not exposing your Home Assistant to the world. But after spending an extended time trying to ensure my network is as secure as possible, giving Home Assistant access to a nodes host network seems pretty risky. One vulnerability in Home Assistant and now someone has access to a lot more than just my smart lights.  

## Configuring the network
My network is split up into multiple networks but for the sake of this post we'll keep it simple and focus on the two that matter; IoT devices on VLAN 1 and the server network on VLAN 2. My firewall is configured to block all inter-VLAN communication. Why? Because I have a lot of smart device from all sorts of random places, and do I trust them? No, and nor should you. However, Home Assistant is no use to us if it can't actually see any devices, so we need an override rule to allow the Home Assistant instance to talk to the IoT network, but not the other way around.

<img src="https://library.wamphlett.net/photos/blog/homelab/home-assistant-firewall-rule.png?w=1080" />

So now Home Assistant can talk directly to the IoT devices, but its still not discovering new devices yet.

## Device discovery
Home Assistant relies on mDNS to find devices on your network, but by default, mDNS only propagates on the network where the device is connected. Well, we have put the devices on a different network to our Home Assistant instance so this is why Home Assistant cant see these devices. This can be resolved by using an mDNS repeater. In my case, Unifi has one built right in so I can tell it to repeat the mDNS packets from the IoT network on VLAN 1 to the servers on VLAN 2. 

<img src="https://library.wamphlett.net/photos/blog/homelab/mdns-settings.png?w=1080" />

We also have to update the firewall to allow mDNS packets across VLANs. I have done this by creating a port group for `5353` (the port mDNS uses)

<img src="https://library.wamphlett.net/photos/blog/homelab/mdns-ip-group.png?w=1080" />

Then I used that port group in my new firewall rules

<img src="https://library.wamphlett.net/photos/blog/homelab/mdns-firewall-rules.png?w=1080" />

We can now use a tool such as `avahi-browse` to validate that the DNS is propagated to our K3S node. 

```shell
avahi-browse -a
+  ens18 IPv6 Presence-Sensor-FP2-1446                      _hap._tcp            local
+  ens18 IPv4 Philips Hue HomeKit                           _hap._tcp            local
```

Perfect. Now if we were happy using the host network, there is nothing more for us to do but as I said previously, I'm not comfortable with that.

## Avoiding the host network
This part of the puzzle had me on a wild goose chase for a long time. Endless hours spent staring at Wireshark and trawling forum posts trying to figure out how I can poke the mDNS records into the Home Assistant pod without giving it access to the hosts network. 

Well thanks to user [lddubeau](https://community.home-assistant.io/u/lddubeau) over on the Home Assistant community forums, [he figured out](https://community.home-assistant.io/t/containers-avoiding-privileged-and-host-network-as-much-as-possible/60792/7) that if you set up an Avahi daemon on the host, you can repeat these devices to the kubernetes/docker network. So thats what I did, I installed Avahi on each of my K3S nodes where Home Assistant runs:

```shell
sudo apt-get update 
sudo apt-get install avahi-daemon
```

Edited the config `/etc/avahi/avahi-daemon.conf` to include a reflector

```shell
[reflector]
enable-reflector=no
reflect-ipv=no
```

Restarted the daemon

```shell
sudo systemctl restart avahi-daemon
```

And thats it, Home Assistant now can see the devices it couldn't see before and it doesn't need access to the host network. 

<img src="https://library.wamphlett.net/photos/blog/homelab/discovered.jpg?w=1080" />

## Making the instance available
Now that Home Assistant is managing devices, we can make make it available. I use [Traefik](https://traefik.io/traefik/) with automatic TLS certificates for network ingest. I'm not going to go into detail about how I set this up, there are many better articles for this. 

Using our deployment from earlier with the removed `hostNetwork` option, we can point a service at Home Assistant.

```yaml
kind: Service
apiVersion: v1
metadata:
  name: home-assistant

spec:
  selector:
    app: home-assistant
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8123
```

And finally define an Ingress.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: home-assistant-ingress
  annotations:
    kubernetes.io/ingress.class: traefik
    cert-manager.io/cluster-issuer: letsencrypt-cloudflare
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"

spec:
  rules:
    - host: "my-home-assistant.com"
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: home-assistant
              port:
                name: http
              
  tls:
    - secretName: home-assistant-tls
      hosts:
        - my-home-assistant.com
```

## Summary
This approach might be overkill for most people, but if like me you have reservations about giving Home Assistant access to the host network, hopefully this gives you a way forward. 