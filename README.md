# Homelab

I'm using TrueNAS Scale for my home server (aka homelab). TrueNAS Scale has a
concept of "Apps" which allows one to quickly install different services (e.g.
pihole, plex, etc) to the server through the TrueNAS Scale UI. There are even
more applications when using [TrueCharts][5].

I was using [TrueCharts][5] happily until they made a backward-incompatible
change which meant that I had to reinstall all the apps. Because of using
PVC's this was very annoying. I really didn't want to do this.

I decided that I didn't want to rely on TrueNAS and TrueCharts charts anymore.
Under the hood, both are just applying [Helm][2] charts. I don't really require
a nice UI and would actually prefer keeping my things in code. This is how this
repository was born.

Another benefit is that if I decide not to use TrueNAS Scale anymore then I
just need a server that has a Kubernetes cluster to redeploy everything.

I'm using [helmfile][1] to manage helm charts. Some of the charts are from
remote repositories. Some of the charts I built myself.

This is not really meant directly for public usage. Everything is configured
for me. I'm also not updating charts and versions very often. I made it public
so that I can put it to GitHub for a backup and for inspiration for people.

## Dependencies

* [helmfile][1]
* [helm][2]
* [vals][3]
* kubectl

`kubectl` has to be configured so that it can access TrueNAS Scale server. See
[this][4] on how to expose kubernetes API.

## Secrets

For secrets I'm using [vals][1]. This integrates well with helmfile. The
secrets themselves are kept in a file called `secrets.yaml` in the root
directory. This file is not commited to the git repository.

## Usage

To see what has changed before applying the changes:

```
helmfile --kube-context=homelab diff .
```

To install (apply) everything run:

```
helmfile --kube-context=homelab apply .
```

[1]: https://github.com/helmfile/helmfile
[2]: https://github.com/helm/helm
[3]: https://github.com/helmfile/vals
[4]: https://gist.github.com/indrekj/8a8121b4964a56cdbb5f6f71d3319457
[5]: https://truecharts.org/
