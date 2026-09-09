# ADR-001: Banking Data Platform Architecture

## Status

Accepted

## Context

The project requires a realistic multi-source banking data engineering platform that demonstrates modern Azure and Databricks engineering practices without creating excessive PAYG cost.

The platform must support:

- heterogeneous source systems
- incremental ingestion
- data quality
- reconciliation
- historical change tracking
- governance
- observability
- CI/CD
- controlled cloud cost

## Decision

The platform will use:

- Azure Data Factory for ingestion and orchestration
- Azure Data Lake Storage Gen2 for landing and storage
- Azure Databricks for distributed processing
- Delta Lake for reliable lakehouse tables
- PySpark and Spark SQL for engineering transformations
- dbt for SQL-centric Gold models
- Unity Catalog for governance and access control
- GitHub for version control and CI/CD

The data platform will follow a Bronze, Silver and Gold architecture.

## Rationale

The architecture separates ingestion, engineering, modelling and governance responsibilities.

This avoids forcing one technology to handle every concern.

The architecture is designed to represent production-level banking engineering patterns while allowing the portfolio deployment to remain cost efficient.

## Consequences

Positive:

- clear separation of responsibilities
- scalable design
- strong governance capability
- realistic data engineering workflow
- suitable for incremental and CDC processing
- suitable for CI/CD and automated testing

Trade-offs:

- introduces multiple platform components
- requires configuration management
- requires careful cost control
- production security controls may be documented rather than fully deployed in the portfolio environment