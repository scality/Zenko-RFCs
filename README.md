# Citadel
Repository of requirements, design documents for Scality S3 and Zenko projects

[https://scality.github.io/citadel/](https://scality.github.io/citadel/)

Requirements

## Structure of this Repository

This repository contains requirements and design for Scality S3 and Zenko.
They follow a template:

- Clear title and lowercase-dashed name
- An overview that contains a description of the problem that we want to solve
- One or more use cases, with a clear mention of the actors in each use case
- Then a detailed description of the technical implementation
- A review of alternative approaches to solve the
  problem. This will show that the proposal has been analyzed at depth
  (think of this as bibliographic reference)

## Contributing
1. Create a python virtual environment: `python -m venv venv`
1. Activate virtual environment: `source venv/bin/activate`
1. Install dependencies: `pip install -r requirements.txt`
1. Start local live reload server: `mkdocs serve`
1. Create or Edit files in "/docs"
1. Copy "TEMPLATE.md" to create a new page
1. Open a pull request

## Help Links
1. [MkDocs](https://www.mkdocs.org/) is the static site generator
1. [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/getting-started/) is the theme and custom options
1. [Mermaid JS](https://mermaid-js.github.io/mermaid/#/) can be used for diagrams and flowcharts
