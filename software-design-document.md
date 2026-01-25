# Postcard - Software Design Document

## Introduction

### Project Overview

Postcard is a web-based platform that enables users to document and share their travel experiences in real-time with selected audiences. The platform allows travelers to create dedicated “trips” representing their holidays, add various types of posts and updates during their journey, and share these experiences with viewers through secure, passcode-protected links.

### Goals

- Provide an intuitive platform for travelers to document their journeys as they happen
- Enable controlled sharing of travel content with different audience tiers (general viewers and close friends)
- Offer a visual, calendar-based interface for trip members to manage and view their travel timeline
- Ensure content is ephemeral and automatically managed through a defined trip lifecycle

### Scope

This initial deployment is a proof-of-concept designed for a small user base (10-20 users). The focus is on core functionality and user experience rather than scaling infrastructure. The platform supports:
- User account creation and authentication
- Trip creation and collaborative posting among trip members
- Multi-tier viewing access via passcodes
- Image upload and storage via pre-signed URLs
- Calendar-based visualization of trip timeline and posts
- Automatic trip lifecycle management and archival

### Primary Features

- **Trip Management**: Create trips with custom names, descriptions, and header images. Invite members, and manage their roles.
- **Multi-format Posts**: Support for destination updates, text posts with optional images, and photo dumps.
- **Tiered Access Control**: Two-level passcode system for general viewers and close friends.
- **Calendar Visualisation**: Interactive calendar showing trip duration and posts plotted by event date to enable easy visualisation of the whole journey.
- **Lifecycle Management**: Automatic trip expiration 60 days post-completion, with email notifications and download options.
- **Image Handling**: Secure image upload via Digital Ocean Spaces with pre-signed URLs.

### Documentation Purpose

The purpose of this document is to specify the requirements that the core features of this platform should deliver, and lay out the high-level design intended to achieve this. These have been created as part of the design phase of the application, to be referenced during development and to appraise if the first phase of this project is complete.

The document provides detailed specifications for system architecture, data models, interfaces, components, and user workflows to guide consistent and effective implementation.

## System Architecture

### High-Level Architecture


## Data Design

### Database Schema

### Users Table

### Trips Table

### Trip Memberships Table (user_trips)

### Trip Invitations Table

### Posts Table

### Post Images Table

### User Sessions Table


### Validation and Integrity Rules

**User Validation**
- Email must be valid format and unique in the system
- Password must meet minimum strength requirements (min 8 characters, mixed case, numbers)
- Name cannot be empty

**Trip Validation**
- Trip name is required and limited to 255 characters
- Start date cannot be in the past (at creation time)
- End date must be after or equal to start date
- Trip duration cannot exceed 180 days
- Viewer and close friends passcodes must be different
- Shareable slug must be unique and URL-safe
- At least one admin must exist for every trip (enforced at application level)

**Trip Lifecycle**
- Trips automatically expire 60 days after end_date
- Expired trips set the `expired_at` timestamp
- Background job runs daily to check and mark expired trips
- Once expired, trips become read-only for members and inaccessible to new viewers

**Post Validation**
- Posts can only be created within the trip’s date range (start_date to end_date)
- Event date must fall within the trip’s duration
- Author must be a member of the trip
- Post type determines required fields:
- `destination`: requires location field
- `text`: requires content field, title optional
- `photo_dump`: requires at least one associated image
- Visibility level determines passcode requirements for viewer access

**Membership Validation**
- A user can only be a member of a trip once (enforced by unique constraint)
- Role must be either ‘admin’ or ‘member’
- Cannot remove the last admin from a trip (enforced at application level)
- Only admins can invite new members

**Invitation Validation**
- Invitations expire after 7 days
- Cannot invite a user who is already a member
- Invitation token must be unique and cryptographically secure
- Only trip admins can create invitations

**Referential Integrity**
- Deleting a trip cascades to delete all memberships, posts, and invitations
- Deleting a user sets post author_id to NULL (preserves post history)
- Deleting a post cascades to delete all associated images

## Interface Design

### Authentication Flow

**Session-based Authentication for Members:**
1. User logs in via `/api/auth/login`
2. Server generates JWT token containing user ID and email
3. Token stored in httpOnly cookie or local storage
4. Subsequent requests include token in Authorization header: `Bearer {token}`
5. Server validates token on each request to protected endpoints

**Passcode-based Authentication for Viewers:**
1. Viewer navigates to trip URL: `/trip/:slug`
2. Viewer submits passcode via `/api/trips/:slug/verify-passcode`
3. Server validates passcode against trip record
4. If valid, server generates short-lived viewer token with access level
5. Viewer token used to fetch posts with appropriate visibility filtering
6. Viewer tokens expire after 24 hours or when browser session ends

### CORS and Security

- CORS configured to allow requests from the deployed Next.js domain
- All passwords hashed using bcrypt with salt rounds of 12
- JWT tokens signed with secure secret key
- Passcodes have last 3 digits stored in plaintext as well as the hashed value (they are meant to be shared, not high-security)
- Rate limiting applied to authentication endpoints (5 attempts per 15 minutes)
- Pre-signed URLs for image uploads expire after 1 hour
- Input validation and sanitisation on all endpoints
- SQL injection prevention via parameterized queries

## Component Design

### Authentication Module

**Purpose**: Handles user registration, login, session management, and authorization.

**Key Functions**:
- `registerUser(email, password, name)`: Creates new user account with hashed password
- `authenticateUser(email, password)`: Validates credentials and generates JWT token
- `validateToken(token)`: Verifies JWT token and extracts user information
- `comparePassword(plaintext, hash)`: Verifies password against stored hash

**Dependencies**:
- bcrypt for password hashing
- jsonwebtoken for JWT generation and validation
- Database connection for user queries

**Algorithm**:
- Password hashing uses bcrypt with 12 salt rounds
- JWT tokens include user ID, email, issued-at timestamp, and 7-day expiration

---

### Passcode Verification Module

**Purpose**: Validates viewer passcodes and establishes viewer sessions.

**Key Functions**:
- `verifyPasscode(slug, passcode)`: Validates passcode against trip
- `generateViewerToken(tripId, accessLevel)`: Creates short-lived JWT for viewers
- `getAccessLevel(passcode, trip)`: Determines if passcode grants close_friends access

**Dependencies**:
- JWT library
- Database connection

**Algorithm**:

```
// Psuedocode
if passcode == trip.viewer_passcode:
  return 'all_viewers'
else if passcode == trip.close_friends_passcode:
  return 'close_friends'
else:
  throw InvalidPasscodeError
```

**Security**:
- Viewer tokens expire after 24 hours
- Tokens are stateless and contain only tripId and accessLevel
- Rate limiting prevents passcode brute-forcing

---

## User Interface Design

### Key User Workflows