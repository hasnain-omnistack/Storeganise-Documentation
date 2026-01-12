# Storeganise Integration Documentation

This document describes how **Storeganise Unit Types** are fetched and displayed on the StorHub website, including the full request flow from frontend to Payload API and finally to the Storeganise external API.

---

## 📄 Page Information

**Page URL**

```
http://storhub-sg.localhost:3003/en/storage-facilities/storhub-serangoon#/unit/listing
```

---

## 🔄 Page Load Flow

When the page loads, the following sequence is executed:

### 1. Buildings Collection API

* Fetches the building using the slug:

  ```
  storhub-serangoon
  ```
* Retrieves **unit types** from the `buildings` collection.

### 2. Storeganise Unit Types Fetch

* Uses building and brand information to fetch unit types from Storeganise via a Payload API proxy.

---

## 🧩 Frontend Implementation

**Component Path**

```
src/components/Building/UnitTypeListings/index.tsx
```

**Code Snippet**

```ts
await storeganiseAPI.getUnitTypes(params, brand.slug || '')
```

This function triggers a **Payload CMS API**, which internally forwards the request to the **Storeganise API**.

---

## 🔌 Payload API (Internal Proxy)

**Endpoint**

```
GET /api/booking/stg/admin/unit-types
```

**Full URL Example**

```
http://storhub-sg.localhost:3003/api/booking/stg/admin/unit-types?siteId=64ae05b48d0b730014125874&limit=1000&offset=0&test=1
```

### Query Parameters

| Parameter | Description                 | Example                    |
| --------- | --------------------------- | -------------------------- |
| siteId    | Storeganise site identifier | `64ae05b48d0b730014125874` |
| limit     | Number of records to fetch  | `1000`                     |
| offset    | Pagination offset           | `0`                        |
| test      | Test mode flag              | `1`                        |

---

## 🛠 Storeganise API Handler (Payload)

All Storeganise-related HTTP methods are handled in the following Payload API route:

**File Path**

```
src/app/api/booking/stg/[...path]/route.ts
```

### Supported HTTP Methods

* `GET`
* `POST`
* `PUT`
* `PATCH`
* `DELETE`

This route acts as a **proxy layer** between the frontend and the Storeganise external API.

---

## 🌐 Storeganise External API

**Endpoint**

```
GET /api/v1/admin/unit-types
```

**Full URL**

```
https://storhub-sg-test.storeganise.com/api/v1/admin/unit-types
```

### Query Parameters

| Parameter | Description                 | Example                    |
| --------- | --------------------------- | -------------------------- |
| siteId    | Storeganise site identifier | `64ae05b48d0b730014125874` |
| limit     | Number of records to fetch  | `1000`                     |
| offset    | Pagination offset           | `0`                        |

---

## 📘 Official Storeganise Documentation

* **Admin Unit Types API**
  [https://storhub-sg-test.storeganise.com/api/docs/admin/unit-types](https://storhub-sg-test.storeganise.com/api/docs/admin/unit-types)

---

## 🔐 Authentication & Account Management

### Login Page

**Page URL**

```
http://storhub-sg.localhost:3003/en/account/login
```

#### Payload API

**Endpoint**

```
POST /api/booking/auth/login
```

**Full URL**

```
http://storhub-sg.localhost:3003/api/booking/auth/login
```

**Payload Example**

```json
{
  "email": "hasnain@omnistack.com",
  "password": "10101010",
  "autologin": false
}
```

This API authenticates the user and proxies the request to the Storeganise authentication service.

---

#### Storeganise API

**Endpoint**

```
POST /v1/auth/token?include=customFields,billingTrigger
```

**Headers**

```
Authorization: Basic base64(email:password)
```

Storeganise returns an authentication token along with related customer metadata.

📘 **Official Documentation**
[https://storhub-sg-test.storeganise.com/api/docs/user/auth](https://storhub-sg-test.storeganise.com/api/docs/user/auth)

---

### Signup Page

**Page URL**

```
http://storhub-sg.localhost:3003/en/account/signup
```

#### Payload API

**Endpoint**

```
POST /api/booking/auth/signup
```

**Payload Structure**

```ts
{
  language: string
  firstName: string
  lastName: string
  email: string
  phone: string
  password: string
  companyName?: string
  customFields: {
    customer_type: 'Individual' | 'Corporate'
    business_phone_number?: string
  }
}
```

This API creates a user account and forwards the request to Storeganise.

---

#### Storeganise API

**Endpoint**

```
POST /v1/users?include=customFields,billingTrigger
```

**Payload**

The payload structure is the same as the Payload signup API and includes customer details and custom fields.

📘 **Official Documentation**
[https://storhub-sg-test.storeganise.com/api/docs/user/users](https://storhub-sg-test.storeganise.com/api/docs/user/users)

---

## 🎧 Zendesk User Creation (Post-Signup)

After a successful signup, a Zendesk user is automatically created.

### Payload API

**Endpoint**

```
POST /api/booking/zendesk/user
```

**Payload**

```ts
{
  userId
  firstName
  lastName
  companyName
  userType
  email
  phone
  marketingConsent
}
```

---

### Zendesk API

**Endpoint**

```
POST https://storhub.zendesk.com/api/v2/users
```

**Headers**

```
Authorization: Basic ${process.env.ZENDESK_API_TOKEN}
```

**Payload Example**

```js
{
  name,
  email,
  phone,
  skip_verify_email: true,
  user_fields: {
    'marketing_consent__shsg_': marketingConsent
      ? 'marketingconsent_shsg_yes'
      : 'marketingconsent_shsg_no',

    ...(IS_MALAYSIA && {
      external_id__shmy_: userId,
    }),
    ...(IS_SINGAPORE && {
      external_id__shsg_: userId,
    }),
  },
}
```

Zendesk users are linked to Storeganise customers using region-specific external IDs.

---

## 🧾 Checkout Flow (Unit Booking)

### Checkout Page

**Page URL Pattern**

```
http://storhub-sg.localhost:3003/en/storage-facilities/{building-slug}/checkout/{unit_id}
```

**Example**

```
http://storhub-sg.localhost:3003/en/storage-facilities/storhub-serangoon/checkout/64dcff4f66446b00140b26b2
```

* The last segment of the URL (`64dcff4f66446b00140b26b2`) represents the **Storeganise `unit_id`**.
* This page is accessed when a user clicks **"Book Now"** on a unit type listing.

---

### Initial Data Fetch

On page load, the checkout page performs the following Storeganise API calls:

#### Fetch Unit Types

**API**

```
GET admin/unit-types?siteId={siteId}&limit=1000
```

* `siteId` is derived from the selected building (`building.site_id`)
* Used to validate and hydrate unit-related data during checkout

---

### Existing Order Handling (Jobs API)

If an `orderId` exists in the session or URL, the checkout flow validates the current job state.

#### Fetch Job Details

**API**

```
GET admin/jobs/{orderId}
```

#### Validation Logic

```ts
if (orderId && job?.step !== 'await_confirmOrder') {
  console.log(
    `Job step for JOB_ID: ${orderId} is not await_confirmOrder redirecting to /storage-facilities/${slug}`,
  )
  redirect('/storage-facilities/')
}
```

* If the job is **not** in `await_confirmOrder` state, the user is redirected back to the facilities listing
* This prevents users from resuming invalid or completed checkout sessions

---

### Checkout Steps

The checkout process is driven by a step-based state machine:

```ts
const CHECKOUT_STEPS = {
  SKELETON: 0,
  SIGNUPFORM: 1,
  SELECT_MOVE_IN_DATE: 2,
  SELECT_PREPAY_PERIOD: 3,
  CHECKOUT_PROFILE: 4,
  RENTAL_AGREEMENT_DETAILS: 5,
  BOOKING_PREVIEW: 6, // SG only
  PAYMENT_METHOD: 7, // MY only
  PAYMENT: 8,
  BOOKING_CONFIRMATION: 9,
  SUCCESS: 10,
  ERROR: 11,
}
```

---

### Initial Checkout State

* On initial page load, the checkout starts with:

  * **`SKELETON (0)`** – loading state while APIs resolve
* After data is loaded, the flow proceeds to:

  * **`SELECT_MOVE_IN_DATE (2)`** – first interactive step for the user

---

### Move-In Date Step (SELECT_MOVE_IN_DATE)

During the **Move-In Date** step, multiple backend operations are triggered to validate booking limits, create orders, and generate sales leads.

---

#### Check Existing Rentals for Customer

This API checks how many active or pending rentals exist for the same customer and determines whether further bookings are allowed.

**API**

```
GET /api/booking/units/rentals/check/{ownerId}?currentOrder={orderId}
```

**Example**

```
http://storhub-sg.localhost:3003/api/booking/units/rentals/check/68b7c79baa8a980bab1f3276?currentOrder=6964dd8f52e45047a014dac7
```

**Response Example**

```json
{
  "success": true,
  "data": {
    "ids": [
      "6964dd8f52e45047a014dac7",
      "68c4683c34f4cf18cc14a542",
      "68c2c4fea3050117bbe00118"
    ],
    "isExceedReservationLimit": false,
    "isAllowInputPromoCode": false,
    "disallowNewUnitsBooking": false
  }
}
```

**Purpose**

* Prevents exceeding reservation limits
* Controls promo code eligibility
* Determines if new unit bookings are allowed

---

#### Create Order (Draft)

Once a move-in date is selected, a **draft order** is created in Storeganise.

**API**

```
POST /api/booking/units/orders
```

**Payload Example**

```json
{
  "siteId": "64ae05b48d0b730014125874",
  "unitTypeId": "64dcff5166446b00140b26bc",
  "startDate": "2026-01-12",
  "prepayPeriods": 1,
  "products": {
    "64ee072db51bc200144e7b08": 1
  },
  "submitLater": true,
  "ownerId": "68b7c79baa8a980bab1f3276",
  "billingMethod": "invoice"
}
```

* `submitLater: true` ensures the order remains incomplete until checkout is finalized

---

#### Fetch Unit Rental Details

After order creation, rental details are fetched from Storeganise using the generated rental ID.

**API**

```
GET /api/booking/stg/admin/unit-rentals/{unitRentalId}?include=customFields
```

**Example**

```
http://storhub-sg.localhost:3003/api/booking/stg/admin/unit-rentals/6964dd8f52e45047a014db46?include=customFields
```

---

#### Create Zendesk Booking Lead

At the same step, a booking lead is created in Zendesk for sales and follow-up purposes.

**Payload API**

```
POST /api/booking/zendesk/lead
```

**Payload Example**

```json
{
  "firstName": "Hasnain",
  "lastName": "Shafqat",
  "companyName": null,
  "userType": "Individual",
  "email": "hasnain@omnistack.com",
  "phone": "+92989898988",
  "siteDetails": {
    "name": "Serangoon",
    "id": "64ae05b48d0b730014125874"
  },
  "unitType": {
    "title": { "en": "X Small 22 sq ft" },
    "code": "xs-ccup-22_00",
    "price": 359.77,
    "id": "64dcff4f66446b00140b26b2"
  },
  "leadType": "booking_lead",
  "flowType": "booking",
  "promotion": { "code": "" },
  "orderId": "6964dd8f52e45047a014dac7",
  "unitRentalId": "6964dd8f52e45047a014db46"
}
```

**Purpose**

* Captures booking intent
* Links Storeganise order and rental to Zendesk
* Enables sales and support follow-ups

---

## ✅ Summary

* Unit types are fetched via a **Payload CMS proxy** to Storeganise
* Login and signup are handled through **Payload authentication APIs**
* Storeganise manages authentication tokens and user records
* Zendesk users are automatically created after signup
* Payload acts as a centralized integration layer for external services

---
