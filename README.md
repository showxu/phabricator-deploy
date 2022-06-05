# phabricator-deploy

## Deploy Phabricator on Docker

- bitnami/phabricator - Docker Image
- bitnami/mariadb - Docker Image

### Use Bitnami Images

- Bitnami closely tracks upstream source changes and promptly publishes new versions of this image using our automated systems.
- With Bitnami images the latest bug fixes and features are available as soon as possible.
- Bitnami containers, virtual machines and cloud images use the same components and configuration approach - making it easy to switch between formats based on your project needs.
- All our images are based on [minideb](https://github.com/bitnami/minideb) a minimalist Debian based container image which gives you a small base container image and the familiarity of a leading Linux distribution.
- All Bitnami images available in Docker Hub are signed with [Docker Content Trust (DCT)](https://docs.docker.com/engine/security/trust/content_trust/). You can use DOCKER_CONTENT_TRUST=1 to verify the integrity of the images.
- Bitnami container images are released daily with the latest distribution packages available.

This [CVE scan report](https://quay.io/repository/bitnami/phabricator?tab=tags) contains a security report with all open CVEs. To get the list of actionable security issues, find the "latest" tag, click the vulnerability report link under the corresponding "Security scan" field and then select the "Only show fixable" filter on the next page.

### Usage

1. Clone this docker compse project.
1. Edit env variables in .env file on demand.
1. Switch to the **correct** docker context for different environment.
1. Run `./staging.sh` for staging environment and `./prod.sh` for production to deploy phabricator containers.

### Troubleshooting

#### This request asked for "/" on host "127.0.0.1", but no site is configured which can serve this request.

set the config parameter `phabricator.base-uri`:

```bash
./bin/config set phabricator.base-uri 'http://example.com/'
```

> **Reference**

- [⚓ T8717 Bad error when first installing phabricator](https://secure.phabricator.com/T8717)
- [apache - This request asked for "/" on host "example.com", but no site is configured which can serve this request - Stack Overflow](https://stackoverflow.com/questions/35628144/this-request-asked-for-on-host-example-com-but-no-site-is-configured-whic)

#### mkdir: cannot create directory '/bitnami/mariadb/data': Permission denied

There are 2 methods to solve the issue:

1. change files(directory) mod, group, owner

```bash
mkdir mariadb-data && sudo chown -R 1001:1001 mariadb-data && sudo chmod -R 775 mariadb-data
docker-compose up
```

For any file or directory, there are 3 types of permissions:

- Read permission
- Write permission
- Execute permission

Using the chmod command, it can set custom permissions to files and directories. Here’s the command structure of any chmod command:

```bash
$ chmod <permission> -R <file_or_directory>
```

Run the `ls -al` command, it’ll print information about the files and directories under current directory. Left column show the encoded file permissions, `drwxr-xr-x` for example. The third column indicates the "user owner" of the file/directory. It’s the user who created this specific file/directory. The fourth column indicates the "group owner". It indicates the user group that has access to the file/directory, any user from the group can access the file/directory.

For a directory, the fisrt character of file permissions will be "d", for a single file, the value will be "-". Total vision:

- Character 1: File (-) or directory (d).
- Character 2-4: Permission for the user owner.
- Character 5-7: Permission for the group owner.
- Character 8-10: Permission for others, for example, users that aren’t the owner and not part of the user group.

In characters 2-10, `r` for read, `w` `for` write, `x` for execute, so the values will come in the form of `rwx`, if a certain value is `-`, then the permission isn’t set. Instead of using characters, it’s also possible to use octal values to signify the permissions, the value ranges from 0 to 7 in octal, 4 for read, 2 for write, 1 for execute, 755 is an octal expression of the permission `rwxr-xr-x`:

- 7 = 4 + 2 + 1: Read, write, and execute (user owner).
- 5 = 4 + 0 + 1: Read and execute permissions (group owner).
- 5 = 4 + 0 + 1: Read and execute permissions (others).

2. use volumes instead of bind mounts

Non-root containers and mounting local folders is problematic, as the permissions are different. Normally, what most users do in this case is to put very open permissions in that mounted folder. Replace ./mariadb-data:/bitnami by mariadb-data:/bitnami (removing ./) in the docker-compose.yml and add the following lines at the end of the file:

```yaml
volumes:
  mariadb-data:
    driver: local
```

> **Reference**

- [mkdir: cannot create directory '/bitnami/mariadb/data': Permission denied · Issue #186 · bitnami/bitnami-docker-mariadb](https://github.com/bitnami/bitnami-docker-mariadb/issues/186)
- [getting "mkdir: cannot create directory '/bitnami/mariadb': Permission denied" · Issue #193 · bitnami/bitnami-docker-mariadb](https://github.com/bitnami/bitnami-docker-mariadb/issues/193)


#### HTTPS setup issues

- not rendering the UI

The problem was that `security.require-https` set to true but phabricator hadn't yet setup HTTPS so it was only using plain old HTTP. In particular, when connect over HTTP to a security.require-https install, phab normally just immediately redirect user to `https://....`

> **Reference**

- [⚓ T12547 Confusing error message when trying to register an account over HTTP with `security.require-https`](https://secure.phabricator.com/T12547)
- [⚙ D17676 Tailor the CSRF check message for HTTP requests with "security.require-https"](https://secure.phabricator.com/D17676)
- [Phabricator installed but not rendering the UI, only bare text/links - ricator - Bitnami Community](https://community.bitnami.com/t/phabricator-installed-but-not-rendering-the-ui-only-bare-text-links/38827/3)

#### SSH issues 

- Connection closed by #{host}, after SSH2_MSG_KEXINIT sent 

First use `ssh -v -T ssh://git@host:2222` to test ssh connection:

```bash
debug1: SSH2_MSG_KEXINIT sent
Connection closed by #{host}
```

It seems the ssh server(sshd) not working properly, try to use `service sshd stop` & `/usr/sbin/sshd -d` (`sudo service ssh stop`, `sudo /usr/sbin/sshd -d`) to debug ssh, the error comes up:

```bash
Permissions 0775 for '/etc/ssh/ssh_host_rsa_key' are too open.
It is required that your private key files are NOT accessible by others.
This private key will be ignored.
Error loading host key "/etc/ssh/ssh_host_rsa_key": bad permissions
Could not load host key: /etc/ssh/ssh_host_rsa_key
```

It turns out that the ssh private key has the excessive file permissions, so change mod 644 to public keys and 600 to private keys:

```bash
chmod 644 #{key.pub}
chmod 600 #{key.private}
```

## Deploy Phabricator on Kubernetes

## Learn more

1. [bitnami/phabricator - Docker Image | Docker Hub](https://hub.docker.com/r/bitnami/phabricator).
1. [bitnami/mariadb - Docker Image | Docker Hub](https://hub.docker.com/r/bitnami/mariadb).
1. [bitnami-docker-mariadb/docker-compose.yml at bitnami/bitnami-docker-mariadb](https://github.com/bitnami/bitnami-docker-mariadb/blob/a2657f6b427d5e38b0a4156aa6e7ea7f3b6d93b5/docker-compose.yml).
1. [Mariadb - Official Image | Docker Hub](https://hub.docker.com/_/mariadb).
1. [Use volumes | Docker Documentation](https://docs.docker.com/storage/volumes/).
1. [Use bind mounts | Docker Documentation](https://docs.docker.com/storage/bind-mounts/).
1. [[bitnami/phabricator] Remove phabricator · Pull Request #7648 · bitnami/charts](https://github.com/bitnami/charts/pull/7648).
1. [Help need with Remote docker port expose · Issue #691 · vmware/photon](https://github.com/vmware/photon/issues/691)