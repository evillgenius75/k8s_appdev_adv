# Extending Kubernetes with Custom Resources

Let’s assume you want to build a clustered application or a software as a service offering. Before you start to write one single line of application code, you must address an overflowing array of architectural issues, including security, multitenancy, API gateways, CLI, configuration management, and logging.

What if you could just leverage all that infrastructure from Kubernetes, save yourself few man years in development, and focus on implementing your unique service?

The latest Kubernetes 1.7 adds an important feature called CustomResourceDefinitions (CRD), which enables plugging in your own managed object and application as if it were a native Kubernetes component. This way, you can leverage Kubernetes CLI, API services, security and cluster management frameworks without modifying Kubernetes or knowing its internals.