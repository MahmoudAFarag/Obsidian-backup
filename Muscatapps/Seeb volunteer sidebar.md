| Section                    | Feature                 | Link                                            |
| -------------------------- | ----------------------- | ----------------------------------------------- |
| **لوحة القيادة والتقارير** | لوحة القيادة            | `/super_admin_dashboard/dashboard`              |
|                            | تحليلات التبرعات        | `/super_admin_dashboard/analytics`              |
|                            | تقرير إجمالي التبرعات   | `/super_admin_dashboard/total-donations-report` |
| **البرامج والتبرعات**      | إعدادات بوابة الدفع     | `/super_admin_dashboard/payment_keys`           |
|                            | الحسابات البنكية        | `/super_admin_dashboard/bank_accounts`          |
|                            | تصنيفات البرامج         | `/super_admin_dashboard/categories`             |
|                            | البرامج                 | `/super_admin_dashboard/services`               |
|                            | الحملات                 | `/super_admin_dashboard/donation-projects`      |
|                            | التبرعات                | `/super_admin_dashboard/orders`                 |
|                            | التبرعات الدورية        | `/super_admin_dashboard/recurring-donations`    |
| **حصالات الخير**           | حصالات الخير الرقمية    | `/super_admin_dashboard/digital-savings-goals`  |
|                            | طلبات الحصالة الميدانية | `/super_admin_dashboard/money_boxes`            |
|                            | المحافظات               | `/super_admin_dashboard/provinces`              |
|                            | الولايات                | `/super_admin_dashboard/states`                 |
|                            | القرى                   | `/super_admin_dashboard/villages`               |
| **الإهداء**                | عمليات الإهداء          | `/super_admin_dashboard/gift-card-requests`     |
|                            | أنواع الإهداء           | `/super_admin_dashboard/gift_types`             |
|                            | بطاقات الإهداء          | `/super_admin_dashboard/cards`                  |
|                            | فئات الإهداء            | `/super_admin_dashboard/donation_categories`    |
| **التطوع والتواصل**        | طلبات التطوع            | `/super_admin_dashboard/volunteers`             |
|                            | الإشعارات               | `/super_admin_dashboard/notifications`          |
|                            | تواصل معنا              | `/super_admin_dashboard/contact_us`             |
|                            | النشرة البريدية         | `/super_admin_dashboard/newsletter`             |
| **محتوى الموقع**           | شرائح الصفحة الرئيسية   | `/super_admin_dashboard/sliders`                |
|                            | البنرات الإعلانية       | `/super_admin_dashboard/promo-banners`          |
|                            | الإنجازات               | `/super_admin_dashboard/achievements`           |
|                            | الإعلانات               | `/super_admin_dashboard/ads`                    |
|                            | الأخبار                 | `/super_admin_dashboard/news`                   |
|                            | الأسئلة الشائعة         | `/super_admin_dashboard/common_questions`       |
|                            | الصفحات القانونية       | `/super_admin_dashboard/legal-pages`            |
| **إعدادات النظام**         | الإعدادات               | `/super_admin_dashboard/settings`               |

## Operational notes before the client demo

- **`digital_goal_holding_service_id` is empty** — set it in الإعدادات (the program that receives digital money-box funds), or the حصالة الخير الرقمية flow can't collect payments.
- The manual statistics (families, quarterly distributions, volunteers) are entered under الإعدادات → إحصائيات المجتمع; the فك الكرب count is auto-calculated from active campaigns flagged as hardship cases.
- The refund policy page is seeded **unpublished with empty content** — write its content in الصفحات القانونية before publishing.
- The root `lang/` folder is now dead weight (everything active lives in `resources/lang/`) — worth deleting in a cleanup commit to avoid future confusion. Same for `lang/ar/ar.json` (54KB of translations that have never loaded anywhere).

Here's the complete lifecycle of حصالة الخير الرقمية as the system works now, actor by actor:

## Phase 0 — One-time setup (admin, before launch)

1. In **البرامج** (`/super_admin_dashboard/services`) create a holding program, e.g. "حصالة الخير الرقمية" — active, with a payment key assigned, not pinned to home. This is the container that receives all box money while donors are still saving.
2. In **الإعدادات** (`/super_admin_dashboard/settings`) → حصالة الخير الرقمية, enter that program's ID in **برنامج استلام مبالغ الحصالة الرقمية**.

Until this is done, donors get _"برنامج الاحتفاظ بأموال الحصالة غير مهيأ"_ when trying to create a box.

## Phase 1 — Donor creates a box

A logged-in donor opens `/#/digital-money-box` in the app, gives the box a **name** and a **target amount** (e.g. "حصالة رمضان — 100 ر.ع"). One active box per donor. Status: `active`.

## Phase 2 — Donor fills the box over time

Whenever they want, the donor contributes any amount up to the remaining balance. Each contribution:

- Creates a real order and goes through **Thawani checkout** using the holding program's payment key → the money physically lands in the bank account behind that key.
- On payment success, `collected_amount` increases; the app shows collected / remaining / progress %.
- Failed or abandoned payments are marked failed and don't count.

During this phase the admin can watch every box at **حصالات الخير الرقمية** (`/super_admin_dashboard/digital-savings-goals`) — donor, target, progress, contribution history. No action needed.

## Phase 3 — Target reached (automatic)

The moment a paid contribution hits the target:

- Status flips to `completed_pending_allocation` — no further contributions accepted.
- The donor automatically gets an **in-app notification + WhatsApp message**: "اكتمل هدف حصالتك — اختر البرنامج الذي ترغب بتخصيص المبلغ له."

## Phase 4 — Donor allocates the money

The donor now sees **all active programs and campaigns** (this is what we just changed — previously only same-payment-key ones) and picks one. The system then:

- Creates a **Transfer** record: holding program → chosen destination, for the full collected amount.
- Writes two offsetting paid orders (−X on holding, +X on destination), so the destination program is **immediately credited in every report**, and the holding program nets back to zero.
- Marks the box `allocated` — the donor's journey is finished. Their history shows the box and where it went.

## Phase 5 — Bank settlement (admin/accountant, only when needed)

This is the new part, at **تحويل الحسابات** (`/super_admin_dashboard/transfers`, now in the sidebar):

- **Destination on the same payment key as the holding program** → the real money is already in the right bank account. The transfer shows **"لا تلزم"** — nothing to do.
- **Destination on a different payment key** → the app's books say the destination program has the money, but the cash is physically sitting in the holding key's bank account. The transfer shows **"بانتظار التسوية"**. The accountant makes a real bank transfer of that amount between the two accounts, then clicks **تأكيد التسوية** — the row turns to "تمت التسوية" with the date and who confirmed it.

## The picture in one line per actor

- **Donor**: create box → pay into it gradually → get notified at 100% → choose any program → done.
- **App**: charges via the holding program's gateway, auto-tracks progress, auto-notifies, credits the destination instantly in all reports via offsetting transfer orders.
- **Accountant**: watches تحويل الحسابات; every "بانتظار التسوية" row = one real bank transfer to execute and confirm. The pending list is exactly the amount by which the banks and the books temporarily disagree.

One practical tip for your client: since most programs and all campaigns sit on key 3, putting the holding program on **key 3** minimizes how often settlement is needed — most allocations will then be "لا تلزم" automatically.