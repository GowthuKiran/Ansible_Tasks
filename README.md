# Ansible_Tasks

---
- name: Certificate Renewal Playbook
  hosts: all
  become: true
  tasks:

    # 1. Taking an existing certificate backup
    - name: Backup existing certificate
      copy:
        src: "/etc/ssl/certs/{{ server_name }}.crt"
        dest: "/backup/certs/{{ server_name }}_backup_{{ ansible_date_time.iso8601 }}.crt"
        remote_src: yes
      tags:
        - backup

    # 2. Add server-specific URL check
    - name: Verify if the server URL is reachable
      uri:
        url: "https://{{ server_name }}:443"
        method: GET
        status_code: 200
        timeout: 10
      register: url_check
      failed_when: url_check.status != 200
      tags:
        - url_check

    # 3. App Stop (Example: stopping a web application)
    - name: Stop the application
      service:
        name: "{{ app_service_name }}"
        state: stopped
      tags:
        - app_stop

    # 4. Stopping any services/instances (e.g., application or database services)
    - name: Stop application-related service
      service:
        name: "{{ service_name }}"
        state: stopped
      tags:
        - stop_services

    # 5. Load Balancer checks (AVX/AVI) and moving the server out of the pool
    - name: Remove server from Load Balancer pool
      uri:
        url: "https://avi-load-balancer/api/pool/{{ lb_pool_name }}/members/{{ ansible_hostname }}"
        method: DELETE
        headers:
          Authorization: "Bearer {{ avi_api_token }}"
      tags:
        - lb_check

    # 6. Stopping PCF instance/process (Assuming BOSH CLI or PCF service management)
    - name: Stop the PCF instance/process
      command: >
        bosh ssh {{ pcf_instance_name }} --command "sudo stop {{ pcf_process_name }}"
      tags:
        - stop_pcf

    # 7. Perform app-specific pre-checks (e.g., Postman/Swagger/Health check)
    - name: Perform pre-checks on application health (using Swagger/Postman)
      uri:
        url: "https://{{ server_name }}/healthcheck"
        method: GET
        status_code: 200
        timeout: 10
      register: app_health_check
      failed_when: app_health_check.status != 200
      tags:
        - app_prechecks

    # Optionally, restart the app after certificate renewal (if required)
    - name: Restart the application after certificate renewal
      service:
        name: "{{ app_service_name }}"
        state: started
      when: app_health_check.status == 200
      tags:
        - app_restart

