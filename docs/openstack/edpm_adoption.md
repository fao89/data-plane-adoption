# EDPM adoption

## Prerequisites

* Previous Adoption steps completed.

## Variables

(There are no shell variables necessary currently.)

## Pre-checks

## Procedure - EDPM adoption

* Create a [ssh authentication secret](https://kubernetes.io/docs/concepts/configuration/secret/#ssh-authentication-secrets) for the EDPM nodes:

  ```
  oc apply -f - <<EOF
    apiVersion: v1
    kind: Secret
    metadata:
        name: dataplane-adoption-secret
        namespace: openstack
    data:
        ssh-privatekey: "{{ edpm_encoded_privatekey }}"
    EOF
  ```

* Stop the nova services.

```bash

# Update the services list to be stopped

ServicesToStop=("tripleo_nova_api_cron.service"
                "tripleo_nova_api.service"
                "tripleo_nova_compute.service"
                "tripleo_nova_conductor.service"
                "tripleo_nova_libvirt.target"
                "tripleo_nova_metadata.service"
                "tripleo_nova_migration_target.service"
                "tripleo_nova_scheduler.service"
                "tripleo_nova_virtlogd_wrapper.service"
                "tripleo_nova_virtnodedevd.service"
                "tripleo_nova_virtproxyd.service"
                "tripleo_nova_virtqemud.service"
                "tripleo_nova_virtsecretd.service"
                "tripleo_nova_virtstoraged.service"
                "tripleo_nova_vnc_proxy.service")

echo "Stopping nova services"

for service in ${ServicesToStop[*]}; do
    echo "Stopping the $service in each controller node"
    $CONTROLLER1_SSH sudo systemctl stop $service
    $CONTROLLER2_SSH sudo systemctl stop $service
    $CONTROLLER3_SSH sudo systemctl stop $service
done
```

* Deploy OpenStackDataPlane:

  ```
  oc apply -f - <<EOF
    apiVersion: dataplane.openstack.org/v1beta1
    kind: OpenStackDataPlane
    metadata:
      name: openstack
    spec:
      deployStrategy:
        deploy: true
      nodes:
        standalone:
          ansibleHost: {{ edpm_node_ip }}
          deployStrategy:
            deploy: false
          hostName: standalone
          node:
            ansibleVars: |
              ctlplane_ip: {{ edpm_node_ip }}
            networks:
            - defaultRoute: true
              fixedIP: {{ edpm_node_ip }}
              name: CtlPlane
              subnetName: subnet1
            - name: InternalApi
              subnetName: subnet1
            - name: Storage
              subnetName: subnet1
            - name: Tenant
              subnetName: subnet1
          role: edpmadoption
      roles:
        edpmadoption:
          deployStrategy:
            deploy: false
          networkAttachments:
            - ctlplane
            - internalapi
            - storage
            - tenant
          preProvisioned: true
          services:
            - configure-network
            - validate-network
            - install-os
            - configure-os
            - run-os
          env:
            - name: ANSIBLE_FORCE_COLOR
              value: "True"
            - name: ANSIBLE_ENABLE_TASK_DEBUGGER
              value: "True"
          nodeTemplate:
            managementNetwork: ctlplane
            ansiblePort: 22
            ansibleSSHPrivateKeySecret: dataplane-adoption-secret
            ansibleUser: root
            ansibleVars: |
              service_net_map:
                nova_api_network: internal_api
                nova_libvirt_network: internal_api

              # edpm_network_config
              # Default nic config template for a EDPM compute node
              # These vars are edpm_network_config role vars
              edpm_network_config_template: templates/single_nic_vlans/single_nic_vlans.j2
              edpm_network_config_hide_sensitive_logs: false
              #
              # These vars are for the network config templates themselves and are
              # considered EDPM network defaults.
              neutron_physical_bridge_name: br-ctlplane
              neutron_public_interface_name: eth0
              role_networks:
              - InternalApi
              - Storage
              - Tenant
              networks_lower:
                External: external
                InternalApi: internal_api
                Storage: storage
                Tenant: tenant

              # edpm_nodes_validation
              edpm_nodes_validation_validate_controllers_icmp: false
              edpm_nodes_validation_validate_gateway_icmp: false

              edpm_ovn_metadata_agent_DEFAULT_transport_url: $(oc get secret rabbitmq-transport-url-neutron-neutron-transport -o json | jq -r .data.transport_url | base64 -d)
              edpm_ovn_metadata_agent_metadata_agent_ovn_ovn_sb_connection: $(oc get ovndbcluster ovndbcluster-sb -o json | jq -r .status.dbAddress)
              edpm_ovn_metadata_agent_metadata_agent_DEFAULT_nova_metadata_host: $(oc get svc nova-metadata-internal -o json |jq -r '.status.loadBalancer.ingress[0].ip')
              edpm_ovn_metadata_agent_metadata_agent_DEFAULT_metadata_proxy_shared_secret: 1234567842
              edpm_ovn_metadata_agent_DEFAULT_bind_host: 127.0.0.1
              edpm_chrony_ntp_servers:
              - clock.redhat.com
              - clock2.redhat.com

              ctlplane_dns_nameservers:
              - $(oc get svc -l service=dnsmasq -o json | jq -r '.items[0].status.loadBalancer.ingress[0].ip')
              dns_search_domains: []
              edpm_ovn_dbs:
              - $(oc get ovndbcluster ovndbcluster-sb -o json | jq -r '.status.networkAttachments."openstack/internalapi"[0]')

              edpm_ovn_controller_agent_image: quay.io/podified-antelope-centos9/openstack-ovn-controller:current-podified
              edpm_iscsid_image: quay.io/podified-antelope-centos9/openstack-iscsid:current-podified
              edpm_logrotate_crond_image: quay.io/podified-antelope-centos9/openstack-cron:current-podified
              edpm_nova_compute_container_image: quay.io/podified-antelope-centos9/openstack-nova-compute:current-podified
              edpm_nova_libvirt_container_image: quay.io/podified-antelope-centos9/openstack-nova-libvirt:current-podified
              edpm_ovn_metadata_agent_image: quay.io/podified-antelope-centos9/openstack-neutron-metadata-agent-ovn:current-podified

              gather_facts: false
              enable_debug: false
              # edpm firewall, change the allowed CIDR if needed
              edpm_sshd_configure_firewall: true
              edpm_sshd_allowed_ranges: ['192.168.122.0/24']
              # SELinux module
              edpm_selinux_mode: enforcing
              plan: overcloud
    EOF
  ```
Note: Role vars will be inherited by nodes, more details [here](https://openstack-k8s-operators.github.io/dataplane-operator/inheritance/)

## Post-checks

* See that ansible jobs are running:

  ```
  while true; do oc logs -f `oc get pods | grep dataplane-deployment | grep Running| cut -d ' ' -f1` 2>/dev/null || echo -n .; sleep 1; done
  ```
