# Authentication Specification

## Metadata / 元数据

| Field | Value |
|-------|-------|
| **Spec ID** | SPEC-002 |
| **Feature** | Local User Authentication |
| **Author** | SDD Tutorial |
| **Created** | 2024-12-01 |
| **Status** | Approved |
| **Priority** | P0-Critical |

---

## Feature Overview / 功能概述

Local authentication provides single-user identity management for SpecTask. Users register a local profile with username and password, then log in to create a session. The session persists across CLI invocations until explicit logout. This is intentionally simple — no network auth, no tokens, no expiration — optimized for the single-developer, local-machine use case.

---

## User Stories / 用户故事

1. As a developer, I want to register a local profile, so that my tasks are associated with my identity.
2. As a developer, I want to log in once and stay logged in across terminal sessions, so that I don't re-authenticate on every command.
3. As a developer, I want to see who is currently logged in, so that I can verify I'm operating as the correct user.

---

## Requirements / 需求 (EARS Format)

### Ubiquitous Requirements / 普适型需求

- **REQ-U01**: The system shall store user profiles with the following fields: id (nanoid), username (string, unique), passwordHash (string), createdAt (ISO 8601 string).

- **REQ-U02**: The system shall store passwords using bcrypt with a cost factor of 12.

- **REQ-U03**: The system shall persist session state in a JSON file at `~/.spectask/session.json` containing the authenticated userId and username.

### Event-Driven Requirements / 事件驱动需求

- **REQ-E01**: When the user executes `spectask register --username <name> --password <pass>`, the system shall create a new user profile with hashed password and display a success message.

- **REQ-E02**: When the user executes `spectask login --username <name> --password <pass>`, the system shall verify the password against the stored hash and, on success, write the session file.

- **REQ-E03**: When the user executes `spectask logout`, the system shall delete the session file and display a confirmation message.

- **REQ-E04**: When the user executes `spectask whoami`, the system shall read the session file and display the current username, or indicate no active session.

### State-Driven Requirements / 状态驱动需求

- **REQ-S01**: While a valid session file exists, the system shall treat the stored userId as the current authenticated user for all task operations.

- **REQ-S02**: While no session file exists or the session file is corrupted, the system shall treat the user as unauthenticated and block task operations.

### Unwanted Behavior Handling / 排除型需求

- **REQ-X01**: If the user attempts to register with a username that already exists, the system shall display: "Username '{name}' is already taken".

- **REQ-X02**: If the user attempts to login with incorrect credentials, the system shall display: "Invalid username or password" (generic message, no hint about which field is wrong).

- **REQ-X03**: If the user attempts to register with a username shorter than 3 characters or longer than 32 characters, the system shall display: "Username must be 3-32 characters".

- **REQ-X04**: If the user attempts to register with a password shorter than 8 characters, the system shall display: "Password must be at least 8 characters".

- **REQ-X05**: If the session file exists but references a userId not in the database, the system shall delete the invalid session file and treat user as unauthenticated.

---

## Acceptance Criteria / 验收标准

### AC-01: Successful registration
- **Given** no user with username "alice" exists
- **When** the user runs `spectask register --username alice --password mypassword123`
- **Then** a new user is created, and output shows "User 'alice' registered successfully"

### AC-02: Successful login
- **Given** user "alice" is registered with password "mypassword123"
- **When** the user runs `spectask login --username alice --password mypassword123`
- **Then** session file is created, and output shows "Logged in as alice"

### AC-03: Session persistence
- **Given** user "alice" is logged in
- **When** the user opens a new terminal and runs `spectask whoami`
- **Then** output shows "Currently logged in as: alice"

### AC-04: Logout clears session
- **Given** user "alice" is logged in
- **When** the user runs `spectask logout`
- **Then** session file is deleted, and output shows "Logged out successfully"

### AC-05: Reject duplicate registration
- **Given** user "alice" already exists
- **When** another registration attempt with "alice" is made
- **Then** operation fails with "Username 'alice' is already taken"

### AC-06: Reject invalid credentials
- **Given** user "alice" is registered
- **When** login is attempted with wrong password
- **Then** operation fails with "Invalid username or password"

---

## Constraints & Assumptions / 约束与假设

### Constraints

- Single user per machine (no multi-session support)
- Password hashing uses bcrypt (cost 12) — acceptable latency for CLI
- Session has no expiration (local tool, physical access = authorization)
- No password recovery mechanism (user can re-register if database is reset)

### Assumptions

- File system at ~/.spectask/ is writable by the current OS user
- bcrypt library is available (native binding via better-sqlite3 build toolchain)
- Terminal can securely accept password input (no echo)

---

## Out of Scope / 范围外

- Multi-user access control or roles
- Password reset or recovery flow
- Two-factor authentication
- OAuth or external identity providers
- Session expiration or refresh tokens
- Account deletion (manual database reset instead)

---

## Dependencies / 依赖

### Upstream Dependencies

| Dependency | Type | Status |
|-----------|------|--------|
| better-sqlite3 | Library | Available |
| bcrypt (or bcryptjs) | Library | Available |
| nanoid | Library | Available |

### Downstream Impact

| Affected System | Impact |
|----------------|--------|
| Task CRUD (SPEC-001) | Requires authenticated userId for all operations |
| All future features | Session validation as prerequisite |
