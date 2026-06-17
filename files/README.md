# Files

## Certificates

To configure SSL certificates for HAProxy, follow these steps:

1. **Obtain SSL Certificates**: Acquire the SSL certificates you want to use. These can be obtained from a Certificate Authority (CA) or generated using tools like OpenSSL.

2. **Prepare Certificates**: Ensure your certificates are in the correct format. HAProxy typically requires PEM format, which includes the private key, the certificate, and any intermediate certificates concatenated together.

3. **Save Certificates**: Place your SSL certificates in the `certs` directory.

4. **Configure HAProxy**: Update your HAProxy configuration file (usually located at `/etc/haproxy/haproxy.cfg`) to use the SSL certificates. Below is an example configuration snippet:

    ```plaintext
    frontend https-in
        bind *:443 ssl crt /etc/haproxy/certs/your-certificate.pem
        mode http
        default_backend servers

    backend servers
        mode http
        server server1 192.168.1.1:80 check
    ```

5. **Reload HAProxy**: After updating the configuration, run the Ansible playbook to restart HAProxy to apply the changes.

For more detailed information on configuring SSL with HAProxy, refer to the official HAProxy documentation: [HAProxy SSL Configuration](https://www.haproxy.com/documentation/hapee/latest/traffic-management/ssl-termination/).

---
> For more information or support, please refer to the official documentations or contact us at info@devopsgroup.sk
