FROM    ubuntu:disco
SHELL   ["/bin/bash", "-euvxo", "pipefail", "-c"]

ARG     OS_ARCH=amd64
ARG     OS_REL=disco
RUN     printf 'hosts: files dns\n' | tee /etc/nsswitch.conf; \
	mkdir -vp /usr/share/man/man1; \
	export DEBIAN_FRONTEND="noninteractive"; \
        apt-get update; \
        apt-get dist-upgrade -y; \
        apt-get install -y --no-install-recommends \
                        apt-transport-https bash ca-certificates cgroupfs-mount curl dnsutils e2fsprogs expect file git git-core gnupg2 jq \
                        less libltdl7 libmagic1 libmagic-mgc libmpdec2 mime-support nano openjdk-11-jdk-headless \
                        openssh-client pigz procps python3-minimal sudo unzip wget xz-utils; \
        export _OS="$(grep -Eio 'ubuntu|debian' /etc/*rel* | cut -d':' -f2 | tr '[[:upper:]]' '[[:lower:]]' | sort | uniq -c | sort -k1,1nr | awk 'NR==1{print $2}')"; \
        curl -fsSLo- "https://download.docker.com/linux/${_OS}/gpg" | apt-key add -; \
        printf "deb [arch=%s] https://download.docker.com/linux/%s %s edge test\\n" "${OS_ARCH}" "${_OS}" "${OS_REL}" | tee /etc/apt/sources.list.d/docker.list; \
        apt-get update; \
        apt-get install -y --no-install-recommends containerd.io docker-ce docker-ce-cli; \
	mkdir -vp /etc/docker; \
	printf '{\n\t"experimental": true\n}\r' | tee /etc/docker/daemon.json; \
        apt-get clean; \
        apt-get autoclean; \
        rm -vrf /var/lib/apt/lists/* /tmp/* /var/tmp/* /var/cache/* ~/.cache

ADD     https://raw.githubusercontent.com/moby/moby/master/hack/dind                            /usr/local/bin/dind
ADD     https://raw.githubusercontent.com/docker-library/docker/master/dockerd-entrypoint.sh    /usr/local/bin/dockerd-entrypoint.sh
RUN     chmod -v a+rx /usr/local/bin/dind /usr/local/bin/dockerd-entrypoint.sh

RUN     addgroup --system dockremap; \
        adduser --system --ingroup dockremap dockremap; \
        printf 'dockremap:165536:65536\n' | tee -a /etc/subuid; \
        printf 'dockremap:165536:65536\n' | tee -a /etc/subgid;

ARG	DOCKER_PORT=2375
ARG	DOCKER_ADDR="127.0.0.1"
EXPOSE	${DOCKER_PORT}
ENV	DOCKER_HOST="tcp://${DOCKER_ADDR}:${DOCKER_PORT}"
VOLUME	/var/lib/docker

ARG     GOPATH="/opt/go"
ARG     GO_VERSION="1.12.6"
RUN     case $OS_ARCH in \
		arm*) export OS_ARCH=armv6l;; \
	esac; \
	curl -fsSLo- "https://dl.google.com/go/go${GO_VERSION}.linux-${OS_ARCH}.tar.gz" | tar zxf - -C /usr/local; \
        printf "export GOPATH=/opt/go\\nexport GOROOT=/usr/local/go\\nexport PATH=\$PATH:\$GOROOT/bin:\$GOPATH/bin\\n" | tee /etc/profile.d/golang.sh; \
        mkdir -vp "${GOPATH}";

ARG     KUBECTL_VERSION
RUN     KUBECTL_VERSION="${KUBECTL_VERSION:-$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)}"; \
	case $OS_ARCH in \
		arm*) export OS_ARCH=arm;; \
	esac; \
        curl -fsSLo /usr/local/bin/kubectl "https://storage.googleapis.com/kubernetes-release/release/${KUBECTL_VERSION}/bin/linux/${OS_ARCH}/kubectl"; \
        chmod -v +x /usr/local/bin/kubectl;

ARG     HELM_VERSION="v2.14.1"
RUN     curl -fsSLo- "https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get" | bash -s - --version "$HELM_VERSION";

ARG     VAULT_VERSION="1.1.3"
RUN     case $OS_ARCH in \
		arm*) export OS_ARCH=arm;; \
	esac; \
	curl -fsSLo- "https://releases.hashicorp.com/vault/${VAULT_VERSION}/vault_${VAULT_VERSION}_linux_${OS_ARCH}.zip" | gunzip -v - > /usr/local/bin/vault; \
        chmod -v +x /usr/local/bin/vault;

ADD	https://raw.githubusercontent.com/jenkinsci/docker-jnlp-slave/master/jenkins-slave	/usr/local/bin/jenkins-slave
ARG     AGENT_VERSION="3.29"
RUN     curl --create-dirs -sSLo /usr/share/jenkins/slave.jar "https://repo.jenkins-ci.org/public/org/jenkins-ci/main/remoting/${AGENT_VERSION}/remoting-${AGENT_VERSION}.jar"; \
        chmod -v a+rx /usr/local/bin/jenkins-slave; \
        chmod -v 755 /usr/share/jenkins; \
        chmod -v 644 /usr/share/jenkins/slave.jar;

ARG     user=jenkins
RUN     export GRPID="$(shuf -i 2000-3000 -n1)"; \
        addgroup --gid "${GRPID}" --system "${user}" ; \
        export USRID="$(shuf -i 3000-4000 -n1)"; \
        useradd  --create-home --home "/home/${user}" --system --shell "/bin/bash" --gid "${GRPID}" --groups docker,sudo --uid "${USRID}" "${user}"; \
        mkdir -vp "/home/${user}/.ssh"; \
        printf 'StrictHostKeyChecking no\n' > /home/${user}/.ssh/config; \
        chmod -v 600 /home/${user}/.ssh/*; \
        printf '[user]\n\tname = %s\n\temail = %s\n' "${user}" "${user}@dind.local" | tee /home/${user}/.gitconfig; \
        printf '%s ALL=(ALL) NOPASSWD: ALL\n' "${user}" >> /etc/sudoers; \
        mkdir -vp "/home/${user}/.jenkins"; \
	chown -vR "${user}:${user}" "/home/${user}" "${GOPATH}"

RUN	printf '#!/bin/bash -evx\n/bin/bash -c "sudo /usr/local/bin/dockerd-entrypoint.sh dockerd --host=tcp://${DOCKER_ADDR:-"127.0.0.1"}:${DOCKER_PORT:-"2375"}" &\n/usr/local/bin/jenkins-slave ${@}\n' | tee /usr/local/bin/entrypoint.sh; \
	chmod -v a+rx /usr/local/bin/entrypoint.sh

ENV     AGENT_WORKDIR="/home/${user}/.jenkins" \
        GOPATH="${GOPATH}" \
        PATH="/usr/local/go/bin:${GOPATH}/bin:${PATH}" \
        LC_ALL="C.UTF-8" \
        LANG="C.UTF-8"

USER    "${user}"
WORKDIR "/home/${user}"

VOLUME  "${AGENT_WORKDIR}"

SHELL   ["/bin/bash", "-c"]
ENTRYPOINT ["entrypoint.sh"]
