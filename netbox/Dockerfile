FROM netbox-community/netbox:v4.5.6

# Habilitar ensurepip e instalar los 9 plugins vía PyPI
RUN python -m ensurepip && \
    pip install --no-cache-dir \
        netbox-custom-objects \
        netbox-validity \
        netbox-lists \
        netbox-inventory \
        netbox-attachments \
        netbox-topology-views \
        netbox-reorder-rack \
        netbox-secrets \
        netbox-plugin-dns