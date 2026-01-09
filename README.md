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

## ✅ Summary

* Frontend loads building data using a **slug**
* Unit types are fetched via a **Payload CMS proxy API**
* Payload forwards the request to the **Storeganise Admin API**
* Storeganise returns unit type data for rendering
