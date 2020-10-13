FROM fedora:32
ENV OCP_CLIENT_URL  https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/openshift-client-linux.tar.gz
ENV RELEASE_VERSION v1.0.1
ENV DOWNLOAD_URL https://github.com/operator-framework/operator-sdk/releases/download/${RELEASE_VERSION}

USER root
# install oc command
RUN curl ${OCP_CLIENT_URL} -o /tmp/openshift-client-linux.tar.gz \
  && tar xvzf /tmp/openshift-client-linux.tar.gz -C /tmp \
  && mv /tmp/oc /usr/local/bin/oc \
  && mv /tmp/kubectl /usr/local/bin/kubectl \
  && rm /tmp/openshift-client-linux.tar.gz /tmp/README.md

# install operator-sdk
RUN curl -L ${DOWNLOAD_URL}/operator-sdk-${RELEASE_VERSION}-x86_64-linux-gnu -o /usr/local/bin/operator-sdk \
  && chmod +x /usr/local/bin/operator-sdk

# install ansible-operator
RUN curl -L ${DOWNLOAD_URL}/ansible-operator-${RELEASE_VERSION}-x86_64-linux-gnu -o /usr/local/bin/ansible-operator \
  && chmod +x /usr/local/bin/ansible-operator

RUN dnf install -y make ansible git vim python-ansible-runner python

RUN pip install openshift ansible-runner-http

USER 1001
