Converting ADF Pipeline as an infrastructure as code for lift and shift architecture

Initial Azure setup of Resource Group, Storage Account, SQL DB and Azure Data factory before deploying ADF. Please run this first and then clone adf_publish repository to setup ADF Pipeline.

terraform {
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.0"
    }
  }
}

provider "azurerm" {
  features {}
}

# ─────────────────────────────────────────
# VARIABLES — client fills these in
# ─────────────────────────────────────────
variable "project_name" {
  description = "Base name for all resources"
  type        = string
  default     = "columbusbank"
}

variable "sql_admin_username" {
  description = "SQL Server admin username"
  type        = string
  default     = "sqladmin"
}

variable "sql_admin_password" {
  description = "SQL Server admin password"
  type        = string
  sensitive   = true
}

# ─────────────────────────────────────────
# 1. RESOURCE GROUP
# ─────────────────────────────────────────
resource "azurerm_resource_group" "rg" {
  name     = "${var.project_name}-rg"
  location = "South Central US"
}

# ─────────────────────────────────────────
# 2. STORAGE ACCOUNT (staging area)
# ─────────────────────────────────────────
resource "azurerm_storage_account" "staging" {
  name                     = "${var.project_name}staging"  # must be globally unique, lowercase, max 24 chars
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_container" "staging_container" {
  name                  = "staging"
  storage_account_name  = azurerm_storage_account.staging.name
  container_access_type = "private"
}

# ─────────────────────────────────────────
# 3. AZURE SQL DATABASE
# ─────────────────────────────────────────
resource "azurerm_mssql_server" "sql_server" {
  name                         = "${var.project_name}-sqlserver"
  resource_group_name          = azurerm_resource_group.rg.name
  location                     = azurerm_resource_group.rg.location
  version                      = "12.0"
  administrator_login          = var.sql_admin_username
  administrator_login_password = var.sql_admin_password
}

resource "azurerm_mssql_database" "sql_db" {
  name      = "${var.project_name}-db"
  server_id = azurerm_mssql_server.sql_server.id
  sku_name  = "S0"  # basic tier, client can upgrade
}

# Allow Azure services to access SQL Server (needed for ADF)
resource "azurerm_mssql_firewall_rule" "allow_azure" {
  name             = "AllowAzureServices"
  server_id        = azurerm_mssql_server.sql_server.id
  start_ip_address = "0.0.0.0"
  end_ip_address   = "0.0.0.0"
}

# ─────────────────────────────────────────
# 4. AZURE DATA FACTORY
# ─────────────────────────────────────────
resource "azurerm_data_factory" "adf" {
  name                = "${var.project_name}-adf"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  identity {
    type = "SystemAssigned"  # lets ADF authenticate to other Azure services
  }
}

# ─────────────────────────────────────────
# OUTPUTS — shown after terraform apply
# ─────────────────────────────────────────
output "resource_group_name" {
  value = azurerm_resource_group.rg.name
}

output "adf_name" {
  value = azurerm_data_factory.adf.name
}

output "sql_server_name" {
  value = azurerm_mssql_server.sql_server.name
}

output "sql_connection_string" {
  value     = "Server=tcp:${azurerm_mssql_server.sql_server.fully_qualified_domain_name},1433;Database=${azurerm_mssql_database.sql_db.name};User ID=${var.sql_admin_username};Password=<your_password>;Encrypt=true;"
  sensitive = true
}

output "storage_account_name" {
  value = azurerm_storage_account.staging.name
}

output "staging_container_name" {
  value = azurerm_storage_container.staging_container.name
}
