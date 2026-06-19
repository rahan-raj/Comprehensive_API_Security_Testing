# API Inventory — crAPI
**Tester:** Rahan Raj K R  
**Date:** June 2026  
**Target:** crAPI running on localhost:8888  

## Authentication Endpoints

| Endpoint | Method | Auth Required | Parameters | Description |
|----------|--------|---------------|------------|-------------|
| /identity/api/auth/login | POST | No | email, password | User login |
| /identity/api/auth/signup | POST | No | name, email, number, password | Registration |
| /identity/api/auth/forget-password | POST | No | email | Password reset |

## User Endpoints

| Endpoint | Method | Auth Required | Parameters | Description |
|----------|--------|---------------|------------|-------------|
| /identity/api/v2/user/dashboard | GET | Yes (JWT) | - | User dashboard |
| /identity/api/v2/user/videos | GET | Yes (JWT) | - | User videos |
| /identity/api/v2/user/videos/{id} | GET | Yes (JWT) | id | Single video |

## Vehicle Endpoints

| Endpoint | Method | Auth Required | Parameters | Description |
|----------|--------|---------------|------------|-------------|
| /identity/api/v2/vehicle/vehicles | GET | Yes (JWT) | - | List vehicles |
| /identity/api/v2/vehicle/{id}/location | GET | Yes (JWT) | id | Vehicle GPS |

## Community Endpoints

| Endpoint | Method | Auth Required | Parameters | Description |
|----------|--------|---------------|------------|-------------|
| /community/api/v2/community/posts/recent | GET | Yes (JWT) | - | Recent posts |
| /community/api/v2/community/posts/{id} | GET | Yes (JWT) | id | Single post |

## Shop Endpoints

| Endpoint | Method | Auth Required | Parameters | Description |
|----------|--------|---------------|------------|-------------|
| /workshop/api/shop/products | GET | Yes (JWT) | - | List products |
| /workshop/api/shop/orders | POST | Yes (JWT) | product_id, quantity | Place order |
| /workshop/api/shop/apply_coupon | POST | Yes (JWT) | coupon_code | Apply coupon |

## Authentication Mechanisms Observed
- JWT Bearer tokens (format: Authorization: Bearer <token>)
- Token appears to use HS256 algorithm (confirmed via jwt.io)
- Tokens observed in: Authorization header

## Notes
- API follows REST conventions
- JSON request/response format throughout
- No API versioning observed on some endpoints