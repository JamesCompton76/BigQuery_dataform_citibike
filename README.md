# BigQuery Dataform Citi Bike Pipeline	

## Overview
A serverless Dataform pipeline built on Google BigQuery to model customer activity and analyze monthly cohort churn using the NYC Citi Bike dataset. This project demonstrates modular analytics engineering, SQLX configuration, and dependency mapping to transform raw telemetry into structured, dashboard-ready data marts.

## Project Structure & Lineage
This pipeline is divided into three distinct layers:
1. **Sources:** Declarations mapping to the raw BigQuery public datasets (`citibike_trips` and `citibike_stations`).
2. **Facts (`fct_customer_monthly_activity`):** A cleansed, aggregated table tracking monthly trip counts, durations, and lifetime boundaries per unique customer surrogate key.
3. **Marts:** Business-facing tables identifying overall monthly churn and segmenting that churn by demographic cohorts (Gender).

**A Note on Source Declarations & Visual Lineage:** 
While Dataform natively supports declaring multiple sources in a single `sources.js` file (which compiles perfectly), local DAG visualizers (like VS Code extensions) often fail to parse `.js` files when rendering the dependency graph. To ensure the raw source tables are accurately reflected in the visual lineage map, the sources in this repository are intentionally declared using individual `.sqlx` files.

## Business Logic: Defining Churn
Tracking churn in a transactional dataset like public transit requires a specific cohort definition, as users do not have explicit "cancellation" dates. 

This pipeline defines the customer states as follows:
* **Active:** A customer is considered active in Month M if they took at least one trip in Month M, *or* if they took exactly a one-month break (active in M-1, zero trips in M, active in M+1).
* **Churned:** A customer is considered churned in Month M if they were active in Month M-1, took zero trips in Month M, and took zero trips in Month M+1.
* **Churn Rate:** Calculated as `Total Churned Customers in Month M / Total Active Customers in Month M-1`.

## Local Development & Installation
This project was developed and compiled locally using the Dataform CLI. 

If running this in a Linux environment (such as WSL), you may need to install the Node.js engine natively to allow the CLI to execute properly.

### 1. Environment Setup
```bash
# Update the package manager and install Node.js/NPM
sudo apt update
sudo apt install nodejs npm -y

# Install the Dataform CLI globally
sudo npm install -g @dataform/cli
```

### 2. Project Initialization
```bash
# Navigate to the project root and install the core Dataform library
npm install
```
*(Note: Requires a `package.json` file in the root directory specifying the `@dataform/core` dependency).*

### 3. Compilation & Testing
To validate the SQLX syntax, resolve dependencies, and ensure the Directed Acyclic Graph (DAG) contains no circular logic, run:
```bash
dataform compile
```

### 4. Visualizing the DAG
To view the visual lineage graph locally without deploying to Google Cloud, open the project in VS Code and install a Dataform DAG extension (e.g., *Dataform DAG* or *Dataform tools*). Use the VS Code Command Palette (`Ctrl + Shift + P`) and execute the "Show Graph" command to render the pipeline map.

## Orchestration & Scheduling
In a production Google Cloud Platform (GCP) environment, this pipeline is designed to be automated rather than manually executed. The `.sqlx` architecture supports multiple scheduling strategies depending on the surrounding infrastructure:

### 1. Native Dataform Scheduling
For a lightweight, serverless approach with zero infrastructure overhead, scheduling is handled directly within the Dataform UI. 
*   **Release Configurations:** Automatically compiles the repository's `main` branch into a deployable state on a defined schedule.
*   **Workflow Configurations:** Executes the compiled release using a standard CRON expression (e.g., `0 0 * * *` for daily runs at midnight).

### 2. Enterprise Orchestration (Cloud Composer / Airflow)
If this pipeline requires coordination with upstream processes (such as a Dataproc PySpark job or an external API ingestion tool), it can be seamlessly orchestrated using Google Cloud Composer. An Apache Airflow DAG can utilize Dataform API operators to trigger the workflow execution only after all raw upstream data has successfully landed in BigQuery.

### 3. Tag-Based Execution at Scale
To efficiently scale the repository as new tables are added, the pipeline utilizes Dataform tags.
*   **Implementation:** Tags are added to the `config {}` block of individual `.sqlx` files (e.g., `tags: ["monthly_churn"]`).
*   **Execution:** Instead of scheduling individual tables, the orchestrator triggers specific tags. Dataform automatically resolves the Directed Acyclic Graph (DAG) and executes only the necessary sources, facts, and downstream marts in the correct dependency order.
