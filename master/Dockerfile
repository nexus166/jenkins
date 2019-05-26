FROM	debian:buster-slim

SHELL	["/bin/bash", "-xeuo", "pipefail", "-c"]

RUN	export JAVA_HOME="/docker-java-home"; \
	printf '#!/bin/sh\nset -e;\n\ndirname "$(dirname "$(readlink -f "$(which javac || which java)")")";\n' | tee /usr/local/bin/docker-java-home; \
        ln -svT "/usr/lib/jvm/java-11-openjdk-$(dpkg --print-architecture)" /docker-java-home; \
        [[ ! -d /usr/share/man/man1 ]] && mkdir -vp /usr/share/man/man1; \
	chmod -v +x /usr/local/bin/docker-java-home; \
	export DEBIAN_FRONTEND=noninteractive; \
        apt-get update; \
        apt-get dist-upgrade -y; \
        apt-get install -y --no-install-recommends \
                bzip2 ca-certificates curl git gnupg2 openjdk-11-jdk-headless openssh-client unzip xz-utils; \
        rm -rf /var/lib/apt/lists/*; \
        apt-get autoclean; \
        [[ "$(readlink -f "$JAVA_HOME")" == "$(/usr/local/bin/docker-java-home)" ]] && update-alternatives --get-selections | awk -v home="$(readlink -f "$JAVA_HOME")" 'index($3, home) == 1 { $2 = "manual"; print | "update-alternatives --set-selections" }'; \
        update-alternatives --query java | tee | grep -q 'Status: manual';

ARG	JENKINS_VERSION
ARG	TINI_VERSION=v0.16.1
ARG	user=jenkins
ARG	group=jenkins
ARG	uid=1000
ARG	gid=1000
ARG	http_port=8080
ARG	https_port=8443
ARG	agent_port=50000
ARG	JENKINS_HOME=/var/jenkins_home

ENV	JENKINS_VERSION="${JENKINS_VERSION:-2.176.1}" \
	JENKINS_HOME="$JENKINS_HOME" \
	JENKINS_SLAVE_AGENT_PORT="${agent_port}" \
	JENKINS_UC="https://updates.jenkins.io" \
	JENKINS_UC_EXPERIMENTAL="https://updates.jenkins.io/experimental" \
	JENKINS_INCREMENTALS_REPO_MIRROR="https://repo.jenkins-ci.org/incrementals" \
	COPY_REFERENCE_FILE_LOG="$JENKINS_HOME/copy_reference_file.log" \
	JAVA_HOME="/docker-java-home" \
	LANG="C.UTF-8"

ARG	JENKINS_URL="https://repo.jenkins-ci.org/public/org/jenkins-ci/main/jenkins-war/${JENKINS_VERSION}/jenkins-war-${JENKINS_VERSION}.war"

ADD	https://raw.githubusercontent.com/jenkinsci/docker/master/plugins.sh            /usr/local/bin/plugins.sh
ADD	https://raw.githubusercontent.com/jenkinsci/docker/master/install-plugins.sh    /usr/local/bin/install-plugins.sh
ADD	https://raw.githubusercontent.com/jenkinsci/docker/master/jenkins-support       /usr/local/bin/jenkins-support
ADD	https://raw.githubusercontent.com/jenkinsci/docker/master/jenkins.sh            /usr/local/bin/jenkins.sh
ADD	https://raw.githubusercontent.com/jenkinsci/docker/master/tini-shim.sh          /bin/tini
ADD	https://raw.githubusercontent.com/jenkinsci/docker/master/tini_pub.gpg          "${JENKINS_HOME}/tini_pub.gpg"

RUN	chmod -v +x \
		/bin/tini \
		/usr/local/bin/install-plugins.sh \
		/usr/local/bin/jenkins-support \
		/usr/local/bin/jenkins.sh \
		/usr/local/bin/plugins.sh; \
	mkdir -vp "$JENKINS_HOME"; \
	chown -vR "${uid}:${gid}" "$JENKINS_HOME"; \
	groupadd -g "${gid}" "${group}"; \
	useradd -d "$JENKINS_HOME" -u "${uid}" -g "${gid}" -m -s /bin/bash "${user}"; \
	curl -fsSL "https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-static-$(dpkg --print-architecture)" -o /sbin/tini; \
	curl -fsSL "https://github.com/krallin/tini/releases/download/${TINI_VERSION}/tini-static-$(dpkg --print-architecture).asc" -o /sbin/tini.asc; \
	gpg --no-tty --import ${JENKINS_HOME}/tini_pub.gpg; \
	gpg --verify /sbin/tini.asc; \
	rm -vrf /sbin/tini.asc /root/.gnupg; \
	chmod -v +x /sbin/tini; \
	mkdir -vp /usr/share/jenkins/ref/init.groovy.d; \
	curl -fsSL "${JENKINS_URL}" -o /usr/share/jenkins/jenkins.war; \
	printf 'import jenkins.model.*\nimport hudson.security.*\n\ndef env = System.getenv()\ndef jenkins = Jenkins.getInstance()\njenkins.setSecurityRealm(new HudsonPrivateSecurityRealm(false))\njenkins.setAuthorizationStrategy(new GlobalMatrixAuthorizationStrategy())\ndef user = jenkins.getSecurityRealm().createAccount(env.JENKINS_USER, env.JENKINS_PASS)\nuser.save()\n\njenkins.getAuthorizationStrategy().add(Jenkins.ADMINISTER, env.JENKINS_USER)\njenkins.save()\n' | tee /usr/share/jenkins/ref/init.groovy.d/init-user.groovy; \
	printf 'import jenkins.model.*\nJenkins.instance.setNumExecutors(0)\n' | tee /usr/share/jenkins/ref/init.groovy.d/no-executors.groovy; \
	chown -vR "${user}:${group}" \
		"$JENKINS_HOME" \
		/usr/share/jenkins/ref \
                /usr/local/bin/install-plugins.sh \
                /usr/local/bin/jenkins-support \
                /usr/local/bin/jenkins.sh \
                /usr/local/bin/plugins.sh

USER    "${user}"

RUN	/usr/local/bin/install-plugins.sh \
		ace-editor ansicolor ant antisamy-markup-formatter apache-httpcomponents-client-4-api artifactory authentication-tokens authorize-project bouncycastle-api \
		branch-api build-keeper-plugin build-pipeline-plugin cloudbees-disk-usage-simple cloudbees-folder conditional-buildstep config-file-provider \
		copyartifact credentials credentials-binding cvs display-url-api docker-commons docker-java-api docker-plugin docker-workflow durable-task \
		email-ext external-monitor-job ghprb git git-client git-server github github-api github-branch-source gradle handlebars http_request icon-shim \
		ivy jackson2-api javadoc jdk-tool job-dsl jobConfigHistory jquery jquery-detached jsch junit kubernetes kubernetes-credentials label-linked-jobs \
		ldap lockable-resources login-theme mailer mapdb-api matrix-auth matrix-project maven-plugin metrics momentjs multiple-scms next-build-number pam-auth \
		parameterized-trigger pipeline-build-step pipeline-graph-analysis pipeline-input-step pipeline-milestone-step pipeline-model-api pipeline-model-declarative-agent \
		pipeline-model-definition pipeline-model-extensions pipeline-rest-api pipeline-stage-step pipeline-stage-tags-metadata pipeline-stage-view pipeline-utility-steps \
		plain-credentials purge-build-queue-plugin rebuild run-condition saferestart scm-api script-security sidebar-link simple-theme-plugin slack ssh ssh-agent ssh-credentials \
		ssh-slaves structs subversion token-macro translation variant windows-slaves workflow-aggregator workflow-api workflow-basic-steps workflow-cps workflow-cps-global-lib \
		workflow-durable-task-step workflow-job workflow-multibranch workflow-scm-step workflow-step-api workflow-support

VOLUME  "$JENKINS_HOME"
EXPOSE  "${http_port}"/tcp "${https_port}"/tcp "${agent_port}"/tcp
ENTRYPOINT ["/sbin/tini", "--", "/usr/local/bin/jenkins.sh"]
