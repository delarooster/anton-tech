---

title: 'The Docker Configuration Anti-Pattern'
type: blog
date: 2025-03-11
prev: blog/container_apps-private_networking.md
next: blog/initial_commit.md
---

# A DevOps Detective Story

*"It works on my machine"* has a cousin, and it's called *"It works in production but nowhere else."* Here's a tale of Docker detective work that might save you hours of debugging frustration.

### The Rescue Mission

Our team was brought in to modernize a containerized application with aging infrastructure. The mission seemed straightforward: migrate the existing images to a more modern environment while making targeted improvements. We aimed to enhance the system while preserving operational stability.

The original implementation had been created some time ago, before many current containerization best practices were established. This is a common scenario in fast-evolving technology stacks.

### The Mystery

During our test environment setup, we encountered an unexpected challenge: despite configuring connections to point to our isolated test database (restored from a production backup), the application persistently connected to the production database.

Our configuration appeared correct. Environment variables were properly set. Yet somehow, the application wasn't respecting these settings.

### The Investigation

After thorough debugging, we had a moment of insight:

*"Could there be hardcoded configuration within the image itself?"*

We ran a targeted search through the container:

```bash
docker exec -it container_name grep -r "connection-string" /
```

The search revealed an `appsettings.json` file embedded at the root of the image with connection information hardcoded.

"Aha!" we thought. "Mystery solved!"

We updated our Kubernetes manifests to override this file... but surprisingly, the application still connected to the original database.

### The Plot Thickens

We expanded our search with a more comprehensive approach:

```bash
docker exec -it container_name find / -name "appsettings.json" -type f
```

What we discovered was illuminating: the application was referencing configuration from two different locations:
- One at the container root (`/home/app/api/appsettings.json`)
- Another within the application files (`/home/app/solution/appsettings.json`)

This created a situation where configuration was being read from different locations depending on which code path executed, leading to unpredictable behavior.

### The Solution

With both configuration locations identified, we implemented a clean solution using Kubernetes volume mounts to consistently override both files:

```yaml
volumeMounts:
  - mountPath: /home/app/api/appsettings.json
    name: config-map-volume
    subPath: appsettings.json
  - mountPath: /home/app/solution/appsettings.json
    name: config-map-volume
    subPath: appsettings.json
```

This approach allowed us to apply our `configmap` containing the appropriate test environment settings to both locations, ensuring consistent configuration throughout the application.

### Lessons Learned

1. **Approach inherited containerized systems methodically.** Container images often contain unexpected configurations that may not be immediately apparent.

2. **Configuration belongs in a) environment variables or b) mounted config files**, never embedded in images. When troubleshooting, recursive searches are invaluable for finding hidden configuration files.

3. **Applications should fail fast with clear error messages** when required configuration is missing. A system that silently falls back to default configurations can create unexpected behavior and potential security concerns.

When building containerized applications, externalize your configuration and implement validation checks for required settings. This approach creates more maintainable, secure, and predictable systems.

---

*Is your organization looking to modernize containerized applications or improve DevOps practices? Our team specializes in optimizing containerization strategies and implementing cloud-native best practices. [Contact us today](https://anton-tech.com/contact/) to discuss how we can help enhance your infrastructure while maintaining operational stability.*