# Pre‑Production Environment Setup for YTLC (CC Schema)
This document describes the complete process to create a new Pluggable Database (PDB) ytlc_preprod_cc inside an Oracle 12c multi‑tenant CDB (enabling), prepare tablespaces, import data from production (schemas: cc, crm, pot, qmdb, jnc, pcc, apig), and perform post‑import validation.

## Table of Contents
- Overview

- Prerequisites

- Step 1: Create the PDB

- Step 2: Create Tablespaces

- Step 3: Create Target Users

- Step 4: Export Data from Production (Already Done)

- Step 5: Import Data into Pre‑Production

- Step 6: Post‑Import Tasks

- Appendix: CDB & PDB Concepts

## Overview
We are setting up a pre‑production environment that mirrors production data for the CC suite of applications. The target is a new PDB ytlc_preprod_cc created inside the existing CDB enabling. Data is transferred via Oracle Data Pump (expdp/impdp) with table‑space remapping where necessary. The process excludes statistics and large log table data to save space.

## Prerequisites
- **Oracle Database 12c (12.2.0.1.0)** with multi‑tenant option.

- Sufficient disk space on /data4 and /data3 (see tablespace sizing).

- Data Pump directory CC_DIR pointing to /data4 (or any shared location accessible by both source and target).

- Export dump files already placed in CC_DIR (see Step 4).

- A user with DBA privilege in the target CDB (e.g. imp_user created automatically during PDB creation).
