# Chefcraft — Weekly Menu Email Template

## Subject Line Options

| # | Subject | Vibe |
|---|---------|------|
| 1 | `Your Chefcraft Menu is Here — Week of {startDate} to {endDate}` | Clean & informative |
| 2 | `What's on the menu this week? 🍽️ {clientName} — {weekRange}` | Friendly & catchy |
| 3 | `This Week's Menu Plan — {clientName} ({templateName})` | Professional |

---

## HTML Email Body Template

```html
<div style="font-family: 'Segoe UI', Arial, sans-serif; max-width: 640px; margin: 0 auto; background: #ffffff; border-radius: 12px; overflow: hidden; border: 1px solid #e8e8e8;">
  
  <!-- Header -->
  <div style="background: linear-gradient(135deg, #00796b 0%, #004d40 100%); padding: 32px 28px; text-align: center;">
    <h1 style="color: #ffffff; margin: 0; font-size: 24px; font-weight: 600;">Chefcraft</h1>
    <p style="color: #b2dfdb; margin: 8px 0 0; font-size: 15px;">Your Weekly Menu Plan</p>
  </div>

  <!-- Greeting -->
  <div style="padding: 28px 28px 16px;">
    <p style="font-size: 16px; color: #333; margin: 0;">Hi <strong>{clientName}</strong> Team,</p>
    <p style="font-size: 15px; color: #555; margin: 12px 0 0; line-height: 1.6;">
      Here's your curated menu for <strong>{weekRange}</strong> ({templateName}). 
      Take a look and let us know if you'd like any tweaks!
    </p>
  </div>

  <!-- Menu Table -->
  <div style="padding: 0 28px 24px;">
    <table style="width: 100%; border-collapse: collapse; font-size: 13px;">
      <thead>
        <tr style="background: #f5f5f5;">
          <th style="padding: 10px 12px; text-align: left; border-bottom: 2px solid #00796b; color: #333; font-weight: 600;">Category</th>
          <th style="padding: 10px 8px; text-align: center; border-bottom: 2px solid #00796b; color: #333; font-weight: 600;">Mon</th>
          <th style="padding: 10px 8px; text-align: center; border-bottom: 2px solid #00796b; color: #333; font-weight: 600;">Tue</th>
          <th style="padding: 10px 8px; text-align: center; border-bottom: 2px solid #00796b; color: #333; font-weight: 600;">Wed</th>
          <th style="padding: 10px 8px; text-align: center; border-bottom: 2px solid #00796b; color: #333; font-weight: 600;">Thu</th>
          <th style="padding: 10px 8px; text-align: center; border-bottom: 2px solid #00796b; color: #333; font-weight: 600;">Fri</th>
          <th style="padding: 10px 8px; text-align: center; border-bottom: 2px solid #00796b; color: #333; font-weight: 600;">Sat</th>
          <th style="padding: 10px 8px; text-align: center; border-bottom: 2px solid #00796b; color: #333; font-weight: 600;">Sun</th>
        </tr>
      </thead>
      <tbody>
        {menuRows}
      </tbody>
    </table>
  </div>

  <!-- Notes (if any) -->
  {notesSection}

  <!-- Footer CTA -->
  <div style="background: #fafbfc; padding: 20px 28px; text-align: center; border-top: 1px solid #eee;">
    <p style="font-size: 14px; color: #666; margin: 0 0 12px;">
      Questions? Reach out to your Chefcraft account manager.
    </p>
    <p style="font-size: 13px; color: #999; margin: 0;">
      Powered by <strong style="color: #00796b;">Chefcraft</strong> — Crafted with care.
    </p>
  </div>

</div>
```

---

## Example Rendered Email (Client: Icon)

**Subject:** `Your Chefcraft Menu is Here — Week of June 23 to June 29`

**From:** noreply@notify.chefcraft.in

---

### Chefcraft
*Your Weekly Menu Plan*

---

Hi **Icon** Team,

Here's your curated menu for **June 23 – June 29, 2026** (**Icon Lunch & Dinner**). Take a look and let us know if you'd like any tweaks!

| Category | Mon | Tue | Wed | Thu | Fri | Sat | Sun |
|----------|-----|-----|-----|-----|-----|-----|-----|
| Salad | Farm Fresh Salad | Green Salad | Caesar Salad | Sprout Salad | Cucumber Salad | Farm Fresh Salad | Green Salad |
| Main Course | Butter Chicken | Palak Paneer | Chicken Curry | Veg Kofta Curry | Methi Matar | Masala | Egg Curry |
| Flavored Rice | Chicken Biryani | Veg Pulao | Channa Pulao | Special Biryani | Veg Pulao | Chicken Biryani | Channa Pulao |
| Steamed Rice | Jeera Rice | Steamed Rice | Jeera Rice | Steamed Rice | Jeera Rice | Steamed Rice | Jeera Rice |
| Indian Bread | Chapathi | Butter Naan | Phulka | Aloo Paratha | Chapathi | Paneer Paratha | Butter Naan |
| Dal/Sambar | Koot Sambar | Dal Tadka | Dal Fry | Dal Tadka (Less Oil) | Koot Sambar | Dal Tadka | Dal Fry |
| Curd/Raita | Curd | Raita | Low Fat Curd | Curd | Raita | Curd | Low Fat Curd |
| Accompaniment | Fryums & Pickle | Gulab Jamun | Fryums & Pickle | — | Fryums & Pickle | Gulab Jamun | Fryums & Pickle |

---

*Questions? Reach out to your Chefcraft account manager.*

*Powered by **Chefcraft** — Crafted with care.*

---

## Placeholder Variables

| Variable | Source |
|----------|--------|
| `{clientName}` | `client.name` (e.g., "Icon", "Mu Sigma") |
| `{weekRange}` | Week start–end dates from selected week |
| `{templateName}` | e.g., "Icon Lunch & Dinner", "Icon Breakfast" |
| `{menuRows}` | Rendered `<tr>` rows from saved menu grid (category × 7 days) |
| `{notesSection}` | Optional block if client notes exist |

## Sending Strategy

- **One email per template per client**
- Icon has templates: ["Icon Lunch & Dinner", "Icon Breakfast"] → **2 emails**
- Each email contains all serving times within that template combined in one table
- Email sent to: `client.user_site_mappings[0].email` or `client.approval_medium`
