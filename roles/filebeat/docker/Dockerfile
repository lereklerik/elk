FROM docker.elastic.co/beats/filebeat:8.0.0
USER root
COPY --chown=root:filebeat filebeat.yml /usr/share/filebeat/filebeat.yml
VOLUME ["/var/lib/docker", "/var/run/docker.sock"]