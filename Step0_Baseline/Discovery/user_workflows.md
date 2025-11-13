# A1 Frontend — Trader Journey Validation

## Scope

Validate the trader flow from **Login → Dashboard → Order placement → Confirmation**, comparing the **A1 frontend design** (images provided) against **expected real-world usage** and **edge cases**.

---

## Images Referenced

| Step | Description            | File                         |
| ---- | ---------------------- | ---------------------------- |
| 1    | Login Page             | [image-1.png](./image/image-1.png) |
| 2    | Select Symbols         | [image-2.png](./image/image-2.png) |
| 3    | Placing Order          | [image-3.png](./image/image-3.png) |
| 4    | Placed Order Confirmed | [image-4.png](./image/image-4.png) |
| 5    | History [Deals]        | [image-5.png](./image/image-5.png) |
| 6    | History [Orders]       | [image-6.png](./image/image-6.png) |

---

## High-Level User Journey

1. **User opens the app** → sees **Login Page** ([image-1.png](./image-1.png))
2. **User authenticates** (credentials / 2FA) → lands on **Dashboard**
3. From dashboard, **user selects symbol(s)** ([image-2.png](./image-2.png))
4. **User opens order modal** / order screen ([image-3.png](./image-3.png)) and fills required fields
5. **User submits order** → receives **Placed Order Confirmed** state ([image-4.png](./image-4.png))
6. **User views status** in **History [Orders]** and executed deals in **History [Deals]** ([image-5.png](./image-5.png), [image-6.png](./image-6.png))

---

## 📌 Notes

* Each step reflects a real-time trading journey for a retail or margin user.
* Flow assumes valid credentials, network stability, and sufficient account balance.
* Variations (e.g., failed order, rejected margin, partial fill) are captured separately in the detailed validation doc.

---
