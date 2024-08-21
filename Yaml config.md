# YAML Configuration Documentation

## Overview

This YAML configuration is used to define monitoring and testing settings for various services under different applications (denoted by EAI numbers). Each EAI number represents a distinct application, and within each application, multiple service modules can be configured. The configuration details include the request parameters, load balancer information, and specific HTTP settings for each service module.

## Structure of the YAML File

### EAI Section

- **`eais`**: This is the root level of the configuration that contains a list of applications, each identified by a unique EAI number.
  - **`eai`**: A unique identifier for an application (EAI number). Replace the placeholder with the actual EAI number.

### Modules Section

Each EAI contains a list of modules. A module corresponds to a specific service within the application that you want to monitor or test.

- **`module_name`**: The name of the service module. This should be a unique identifier for the service within the application.
- **`active`**: A boolean flag indicating whether the module is active (`true`) or inactive (`false`). Inactive modules will be ignored by the monitoring system.
- **`defaults`**: Contains default settings for the service request.
  - **`port`**: The port number to be used (typically `443` for HTTPS).
  - **`protocol`**: The protocol to use (`HTTPS`).
  - **`method`**: The HTTP method to use (`GET`, `POST`, etc.).
  - **`path`**: The API endpoint path.
  - **`expected_response_code`**: The HTTP response code expected from the service (e.g., `200`).
  - **`okta_required`** *(Optional)*: A boolean indicating whether Okta authentication is required.
  - **`body`** *(Optional)*: The JSON payload to be sent in the request body for `POST` requests.
- **`load_balancers`**: A list of load balancers associated with the service module.
  - **`name`**: The hostname of the load balancer.
  - **`dc`**: The data center or environment identifier (e.g., `LB`).

### Example YAML Configuration

```yaml
eais:
  - eai: 1234567  # Example EAI number, replace with actual value
    modules:
      - module_name: service-one
        active: true
        defaults:
          port: 443
          protocol: HTTPS
          method: GET
          path: /api/service-one/resource?parameter=value
          expected_response_code: 200
        load_balancers:
          - name: loadbalancer1.example.com
            dc: LB
      # Additional modules here...
```

### Adding More Applications

To configure additional applications, simply duplicate the entire EAI block (including the `modules` section) and append it below the existing configuration. Ensure that each EAI block has a unique `eai` number.

### Example of Adding a New Application

```yaml
eais:
  - eai: 1234567
    modules:
      # Modules for the first application
  - eai: 7654321  # New application
    modules:
      - module_name: service-one
        active: true
        defaults:
          port: 443
          protocol: HTTPS
          method: GET
          path: /api/service-one/resource?parameter=value
          expected_response_code: 200
        load_balancers:
          - name: loadbalancer21.example.com
            dc: LB
      # Additional modules for the second application...
```

## Key Points

- **EAI Number**: Each application should have a unique EAI number. This acts as the identifier for the application.
- **Active Flag**: Set the `active` flag to `true` for any module you want to monitor or test. Inactive modules are ignored.
- **Load Balancers**: Define at least one load balancer for each service module to specify where the requests should be directed.
- **Duplicating for Multiple Applications**: To configure multiple applications, duplicate the entire `eai` section and modify it for each new application.
