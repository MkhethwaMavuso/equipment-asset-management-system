# equipment-asset-management-system
A robust enterprise equipment and asset management portal built on Retool and PostgreSQL featuring automated prefix tagging, secure bcrypt authentication, and role-scoped inventory tracking.

📋 Table of Contents📌 1. Product Requirements Document (PRD)🗄️ 2. Database Schema & Entity Architecture🏗️ 3. System Architecture & Retool Data Flow🛡️ 4. Security & Role-Based Access Control (RBAC)🔌 5. API Contract Specifications🚀 Setup & Installation📌 1. Product Requirements Document (PRD)❓ The ProblemOperational Inefficiencies: Manual input of hardware identifiers leads to duplicated asset tags, lost inventory, and unaligned records across departments.Security & Data Exposure: Standard employees often have unrestricted access to full inventory valuations, administrative settings, and other users' confidential asset allocations.Lack of Self-Service Transparency: Absence of a centralized, transparent portal for employees to request equipment check-outs, resulting in untracked handoffs.💡 The SolutionA centralized Equipment & Asset Management Portal that automates asset lifecycle tracking, enforces role-scoped visibility, and provides self-service hardware requests backed by server-side PostgreSQL security and PL/pgSQL sequencing.🎯 Key Functional Requirements🔐 FR-1 [Auth & User Management]: Users sign up and log in using custom credentials securely hashed via PostgreSQL pgcrypto (bcrypt).🏷️ FR-2 [Automated Asset Tagging]: The system automatically generates collision-free, category-prefixed asset tags (CMP-001024, MCH-001025, VHC-001026, OFF-001027) upon registration.⚡ FR-3 [Self-Service Equipment Request]: Standard users select equipment categories, choose available inventory items, and submit request notes.📊 FR-4 [Request Fulfillment & Analytics]: Inventory Managers and Admins review pending requests and execute single-click approvals or rejections with instant status reflection.🗄️ 2. Database Schema & Entity ArchitectureThe backend database runs on Retool Database (PostgreSQL), adhering to strict relational constraints and PL/pgSQL sequences for automated identifier generation.📊 Entity-Relationship OverviewTable NamePrimary KeyForeign KeysKey Attributes & Description👤 usersid (SERIAL)Nonefull_name, email (UNIQUE), password_hash, role📦 equipment_assetsid (SERIAL)assigned_to -> users(email)asset_tag (UNIQUE), name, category, status, value📑 equipment_requestsid (SERIAL)asset_id -> equipment_assets(id)requester_email, comments, status⚡ Automated Key Generation (PostgreSQL)SQLCREATE SEQUENCE IF NOT EXISTS asset_tag_seq START WITH 1001 INCREMENT BY 1;

CREATE OR REPLACE FUNCTION generate_asset_tag(category_prefix TEXT DEFAULT 'AST')
RETURNS TEXT AS $$
BEGIN
    RETURN category_prefix || '-' || LPAD(nextval('asset_tag_seq')::TEXT, 6, '0');
END;
$$ LANGUAGE plpgsql;
🏗️ 3. System Architecture & Retool Data FlowThe architecture follows a multi-tier structure separating client presentation, logic transformation, and database persistence.┌──────────────────────────────────────────────────────────────────┐
│                   CLIENT LAYER (Retool Canvas UI)                │
│                                                                  │
│  ┌───────────────────────┐             ┌───────────────────────┐ │
│  │ Navigation Header     │             │ Dynamic Data Grids    │ │
│  │ (Tabbed Layout)       │             │ (Tables & Form Views) │ │
│  └───────────────────────┘             └───────────────────────┘ │
│                                                                  │
│  • Session State Manager: `app_user`                             │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                                 │ Parametrized Data Binding
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│              APPLICATION LOGIC & TRANSFORMATION TIER            │
│                                                                  │
│  • Client JS Transformers: Input Sanitation & RBAC Guards        │
│  • Retool Event Pipelines: Trigger Cascades & Toast Alerts       │
└────────────────────────────────┬─────────────────────────────────┘
                                 │
                                 │ Parameterized SQL Queries
                                 ▼
┌──────────────────────────────────────────────────────────────────┐
│                PERSISTENCE TIER (PostgreSQL Database)            │
│                                                                  │
│  • Security Engine: `pgcrypto` (Bcrypt Password Verification)    │
│  • Sequence Engine: PL/pgSQL Functions                           │
│  • Storage Engine: Relational Tables & Constraints               │
└──────────────────────────────────────────────────────────────────┘
🔄 Data Execution Lifecycle🖱️ User Action: User selects equipment and submits request notes via requestEquipmentForm.⚙️ Client Validation: Retool's frontend evaluates component rules and required fields.🗄️ Query Execution: Retool dispatches a parameterized SQL mutation (submitEquipmentRequest) to PostgreSQL.⚡ Database Atomicity: PostgreSQL inserts a pending request row and updates the asset status atomically.🔔 Event Cascade: On success, a toast notification displays and re-fetches the asset table state (.trigger()).🛡️ 4. Security & Role-Based Access Control (RBAC)🔐 Password Hashing (pgcrypto)All passwords are encrypted server-side using PostgreSQL's pgcrypto module with standard bcrypt blowfish salt rounds (gen_salt('bf', 10)).🎭 Access Control MatrixSystem Feature / ActionStandard UserInventory ManagerAdministrator👁️ View Assigned Assets✅✅✅📂 Browse All Organizational Assets❌✅✅📝 Request Equipment✅✅✅➕ Create New Asset❌✅✅✅ Approve / Reject Requests❌✅✅⚙️ Access Administrative Portal❌✅✅🛡️ Component Security GuardsRetool UI component properties (Hidden, Disabled) evaluate the app_user session variable to enforce interface restrictions dynamically:JavaScript// Hide Action Buttons for non-admin/manager users
{{ !['Admin', 'Inventory Manager'].includes(app_user.value?.role) }}
🔌 5. API Contract Specifications📥 5.1 Asset Registration API (insertAsset)Type: SQL Mutation QueryAccess Level: Inventory Manager & AdminInput Parameters:JSON{
  "category": "Computing | Machinery | Vehicles | Office Hardware",
  "name": "string",
  "value": "number"
}
SQL Implementation:SQLINSERT INTO equipment_assets (asset_tag, name, category, value, status)
VALUES (
  generate_asset_tag(
    CASE {{ addAssetForm.data.categorySelect }}
      WHEN 'Computing' THEN 'CMP'
      WHEN 'Machinery' THEN 'MCH'
      WHEN 'Vehicles' THEN 'VHC'
      WHEN 'Office Hardware' THEN 'OFF'
      ELSE 'AST'
    END
  ),
  {{ addAssetForm.data.nameInput }},
  {{ addAssetForm.data.categorySelect }},
  {{ addAssetForm.data.valueInput }},
  'Available'
)
RETURNING id, asset_tag, name, status;
📤 5.2 Role-Scoped Asset Query (getAssets)Type: Parametrized SQL Read QueryAccess Level: All Active Roles (Filtered by RLS logic)SQL Implementation:SQLSELECT * FROM equipment_assets
WHERE 
  {{ app_user.value.role }} IN ('Admin', 'Inventory Manager')
  OR assigned_to = {{ app_user.value.email }}
ORDER BY id DESC;
🚀 Setup & InstallationDatabase Provisioning:Open your Retool Database console.Run the DDL script to initialize users, equipment_assets, and equipment_requests tables, alongside sequences and generate_asset_tag().App Import:Import the provided application blueprint into your Retool workspace.Session Configuration:Ensure your login query populates the app_user state variable upon successful credential validation.
