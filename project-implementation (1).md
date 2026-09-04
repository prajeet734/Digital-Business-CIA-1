# Project Implementation Tracker — Inventory Management System

This is the group's single, living work log. Update it whenever a task is planned, started, completed, blocked, or reopened. Do **not** wait until submission to fill this in retroactively — evaluators cross-check this file against GitHub history and the actual code.

**Rules for this file:**
- Break work into small, identifiable tasks (never one line like "Build backend").
- `Completed By` must be a student's name — never an AI tool.
- `AI Assistance` = Yes/No/Partial. If AI helped, the responsible student must still verify, test, and be able to explain the implementation in the viva.
- `Evidence` = commit hash, PR link, screenshot, test result, or API response.

## Status values
| Status | Meaning |
|---|---|
| Pending | Task identified but not started |
| In Progress | Started, not complete |
| Completed | Implementation complete and verified |
| Blocked | Cannot proceed — documented dependency/problem |
| Reopened | Previously completed task found to have a problem |

## Work Log

| Task ID | Task | Component | Assigned To | Status | Completed By | Date Completed | AI Assistance | Evidence |
|---|---|---|---|---|---|---|---|---|
| T001 | Design database schema (6+ entities, relationships) | Data Layer | | Pending | | | | |
| T002 | Create Users table + model | Data Layer | | Pending | | | | |
| T003 | Create Products table + model | Data Layer | | Pending | | | | |
| T004 | Create Categories table + model | Data Layer | | Pending | | | | |
| T005 | Create Suppliers table + model | Data Layer | | Pending | | | | |
| T006 | Create StockTransactions table + model | Data Layer | | Pending | | | | |
| T007 | Create PurchaseOrders table + model | Data Layer | | Pending | | | | |
| T008 | Implement user registration API | Backend/Auth | | Pending | | | | |
| T009 | Implement login API (JWT issuance) | Backend/Auth | | Pending | | | | |
| T010 | Implement auth middleware (JWT verify + role check) | Backend/Auth | | Pending | | | | |
| T011 | Implement Products CRUD API | Backend | | Pending | | | | |
| T012 | Implement Suppliers CRUD API | Backend | | Pending | | | | |
| T013 | Implement StockTransactions API (record stock in/out) | Backend | | Pending | | | | |
| T014 | Implement stock quantity update logic on transaction | Backend | | Pending | | | | |
| T015 | Implement reorder-point / priority ranking algorithm | Backend/Algorithm | | Pending | | | | |
| T016 | Implement PurchaseOrders API (create/approve/status update) | Backend | | Pending | | | | |
| T017 | Implement KPI/dashboard aggregation endpoint | Backend | | Pending | | | | |
| T018 | Build staff login + dashboard UI | Frontend | | Pending | | | | |
| T019 | Build stock lookup + record-movement UI | Frontend | | Pending | | | | |
| T020 | Build manager dashboard (KPIs, reorder list) UI | Frontend | | Pending | | | | |
| T021 | Build product/supplier/staff management UI | Frontend | | Pending | | | | |
| T022 | Build purchase order approval UI | Frontend | | Pending | | | | |
| T023 | Connect frontend to backend API (integration) | Integration | | Pending | | | | |
| T024 | Write/test business workflow end-to-end (stock-out → reorder flag → PO) | Testing | | Pending | | | | |
| T025 | Write docs/architecture.md | Documentation | | Pending | | | | |
| T026 | Deploy demo build (frontend + backend + DB) | Deployment | | Pending | | | | |

> Add new rows as tasks are identified. Keep Task IDs sequential. This table is the primary evidence source for individual contribution — keep it accurate.
