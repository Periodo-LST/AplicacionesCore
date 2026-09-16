FROM netboxcommunity/netbox:v4.5.6

USER root

RUN python3 -m pip install --no-cache-dir \
    netboxlabs-netbox-custom-objects \
    netbox-validity \
    netbox-lists \
    netbox-inventory \
    netbox-attachments \
    netbox-topology-views \
    netbox-reorder-rack \
    netbox-secrets \
    netbox-plugin-dns

USER unit