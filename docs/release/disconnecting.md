# Disconnecting Releases

We are currently in a multi-repository environment.
We must redefine how our products are connected to their component repositories, taking how we intend to release products into consideration.


## Component Repositories

We have many libraries, APIs, and tools, which we have coded in-house.
For example, Arsenal, CloudServer, and Backbeat.
Let's treat them as we would any open-source component we use in a project.

Each component must contain:

* Unit and functional tests to ensure that the component works.
* Documentation explaining the services the component provides, and how to contribute, build, test, configure, run, use, and (potentially) monitor the component.
* Release notes
* Build chain for building, testing, and publishing packages

Each component will thus have its own release cycle that is independent of the release cycle of the product itself.
The number of component releases, therefore, does not affect the product release cycle.

Selecting a component should be as easy and understandable as picking one from the open source world either from it’s proper language registry (like, for example, bumping the version on the `package.json` file for Node.js projects) or pulling a Docker image from a registry.
The component must therefore make those packages or images available.

Each component release is made by the project leader with the help of the CI chapter, advertised internally and externally if needed.
Release notes are provided with each release.

## Project Repositories

The purpose of a project repository is to:

* Select components (developed in-house or from open source projects) by version. 
* Create a build chain that outputs the desired installation code and packages.
* Trigger integration tests that ensure all selected components interoperate effectively.
* Document the project's services, and how to build, test, configure, run, monitor, and contribute to the project.
* Offer release notes that document changes in the product's function and features.

Project releases should be kept as small as possible.
For example, bumping the version of a component might unlock features or bugfixes.
If we keep shipping releases, we can advance customers a few at a time and receive needed feedback along the way.

The responsibility for providing instructions and all required packages for installation is at the project level, not the product level.

Release readiness is determined by the project owner. 

## Product Repositories

Product repositories contain the code and documentation that are shipped to the customer.

The purpose of a product repository is to:

* Assemble projects and components, maintaining their respective version numbers.
* Create a build chain that outputs the desired installation code and packages.
* Trigger integration tests that ensure all selected projects interoperate effectively.
* Document the product's services; how to install, configure, run, and monitor the product; and how to solve product issues.
* Offer release notes that document changes in the project's function and features.

Product releases are driven by customer needs and are not defined within the object squad. 
The object squad, however, is responsible for making project releases available to the product and for helping integrate the projects into the product. 
