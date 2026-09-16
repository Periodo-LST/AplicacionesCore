FROM netboxcommunity/netbox:v4.5.6

USER root

RUN /usr/local/bin/uv pip install --python /opt/netbox/venv/bin/python3 --no-cache \
    netboxlabs-netbox-custom-objects \
    netbox-validity \
    netbox-lists \
    netbox-inventory \
    netbox-attachments \
    netbox-topology-views \
    netbox-reorder-rack \
    netbox-secrets \
    netbox-plugin-dns

RUN mkdir -p /opt/netbox/netbox/static/netbox_topology_views/img

USER unit