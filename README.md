# README

## Overview

This project involves creating a 3-tier web API using Docker containers. The application consists of three main components: an HTTP server, a Backend API, and a Database. Each component will be containerized and we will orchestrate these containers to work together seamlessly. The final goal is to have a fully functional web API running with Docker Compose.

## Goals

- Follow good practices for Docker containerization.
- Document each step of the process.
- Create an appropriate file structure with 1 folder per image.
- Successfully run a 3-tier web API.

## Steps

### 1. Database

#### Base Image

We will use the image `postgres:14.1-alpine` to create our database and we will configure it by changing our Dockerfile for logins and creating our dataset:

```Dockerfile
FROM postgres:14.1-alpine

ENV POSTGRES_DB=db \
    POSTGRES_USER=Navid \
    POSTGRES_PASSWORD=1234

COPY 01-CreateScheme.sql /docker-entrypoint-initdb.d
COPY 02-InsertData.sql /docker-entrypoint-initdb.d
