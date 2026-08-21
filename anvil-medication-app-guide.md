# Building the Medication Tracker in Anvil

Your mockups map onto Anvil pretty cleanly — it's a data-table-backed CRUD app with a bit of date logic. Here's how I'd structure it.

## 1. Data model (Data Tables service)

Add the **Data Tables** and **Users** services, then create these tables:

**`medications`**

| column | type |
|---|---|
| `user` | Link to Users |
| `name` | Text |
| `quantity` | Number (pills left in bottle) |
| `frequency` | Text — `"daily"` or `"weekly"` |
| `days_of_week` | Simple Object — e.g. `["Mon","Wed"]` |
| `icon` | Media (the pill images on your Medicine screen) |

**`doses`** — one row per medication per day

| column | type |
|---|---|
| `medication` | Link to medications |
| `date` | Date |
| `taken` | True/False |
| `side_effects` | Text |

**`notifications`**

| column | type |
|---|---|
| `user` | Link to Users |
| `text` | Text |
| `created` | Date and Time |
| `kind` | Text — `"missed"`, `"added"`, `"low_stock"` |

**`medication_catalog`** — the prescription-medication dropdown from mockup 2. Just a `name` column, pre-populated. Custom meds get written straight to `medications` without a catalog link.

The `doses` table is the important design decision: instead of storing "taken today" on the medication itself, you write one row per med per day. That's what makes the calendar view (31/31 this month) and the notification history possible.

## 2. Form structure

- **`MainLayout`** — a Layout (not a Form) holding the bell icon top-right and the bottom nav. Every page uses it as its layout.
- **`Home`** — the medication checklist.
- **`Calendar`** — month grid.
- **`Medicine`** — the icon grid.
- **`Notifications`** — the list.
- **`AddMedicationDialog`** — a plain Form shown via `alert()`.
- **`MedItem`** — the row template for the repeating panel.

## 3. The bottom nav bar

Anvil's M3 theme gives you a top app bar plus a navigation rail or drawer out of the box, but **not** a bottom bar — that's the one piece you'll need to build. Drop a `FlowPanel` at the bottom of `MainLayout`, put three `NavigationLink`s in it, set each one's `navigate_to` to Home / Calendar / Medicine, and give the panel a Role (say `bottom-nav`) with CSS in `theme.css`:

```css
.anvil-role-bottom-nav {
  position: fixed; bottom: 0; left: 0; right: 0;
  display: flex; justify-content: space-around;
  padding: 8px 0; background: var(--md-sys-color-surface-container);
}
```

Because `NavigationLink` handles the routing, you don't write any navigation code — the links know which form is active and highlight themselves.

## 4. Home screen

Drop a `RepeatingPanel` with `MedItem` as its item template. In `Home`:

```python
from ._anvil_designer import HomeTemplate
import anvil.server
import datetime

class Home(HomeTemplate):
  def __init__(self, **properties):
    self.init_components(**properties)
    self.label_date.text = datetime.date.today().strftime("Today, %d/%m/%y")
    self.rp_meds.items = anvil.server.call('get_todays_doses')

  def btn_add_click(self, **event_args):
    from ..AddMedicationDialog import AddMedicationDialog
    if alert(AddMedicationDialog(), large=True, buttons=[("Save", True), ("Cancel", False)]):
      self.rp_meds.items = anvil.server.call('get_todays_doses')
```

Server module:

```python
import anvil.server, anvil.users, datetime
from anvil.tables import app_tables

@anvil.server.callable
def get_todays_doses():
  user = anvil.users.get_user()
  today = datetime.date.today()
  weekday = today.strftime('%a')          # 'Mon', 'Tue', ...
  out = []
  for med in app_tables.medications.search(user=user):
    due = med['frequency'] == 'daily' or weekday in (med['days_of_week'] or [])
    if not due:
      continue
    dose = app_tables.doses.get(medication=med, date=today)
    if dose is None:                       # create the row lazily on first view
      dose = app_tables.doses.add_row(medication=med, date=today, taken=False)
    out.append({'med': med, 'dose': dose})
  return out
```

`app_tables.doses.get()` returns a single row or `None`, so the lazy-create pattern above means you never need a nightly job to seed the day's doses — they appear the first time the user opens the app that day.

And the row template, which handles both the tick box and the expand-for-side-effects behaviour from mockup 3:

```python
class MedItem(MedItemTemplate):
  def __init__(self, **properties):
    self.init_components(**properties)
    self.label_name.text = self.item['med']['name']
    self.cb_taken.checked = self.item['dose']['taken']
    self.panel_side_effects.visible = False

  def cb_taken_change(self, **event_args):
    anvil.server.call('mark_taken', self.item['dose'], self.cb_taken.checked)
    self.panel_side_effects.visible = self.cb_taken.checked

  def btn_save_effect_click(self, **event_args):
    anvil.server.call('log_side_effect', self.item['dose'], self.tb_effect.text)
    self.panel_side_effects.visible = False
```

Wrap the whole row in an M3 `Card` with the panel below the label — toggling `visible` gives you the grow-on-click effect.

## 5. Add Medication dialog

The pieces map directly to M3 components: a `TextField` with `suggestions` (or a `DropDownMenu` fed from `medication_catalog`) for the med picker, a numeric `TextField` for quantity, `RadioButton`s for Daily/Weekly, and a `FlowPanel` of seven toggleable `Button`s for S M T W T F S. Store the selected days as a list on the form and pass it up:

```python
  def get_med_data(self):
    return {
      'name': self.dd_medication.selected_value or self.tb_custom.text,
      'quantity': int(self.tb_quantity.text or 0),
      'frequency': 'daily' if self.rb_daily.selected else 'weekly',
      'days_of_week': self.selected_days,
    }
```

## 6. Calendar

Python's `calendar` module does the grid layout for you — `monthcalendar` returns a list of weeks, with `0` for padding days outside the month:

```python
import calendar, datetime

  def build_month(self, year, month):
    self.fp_days.clear()
    for week in calendar.monthcalendar(year, month):
      for day in week:
        if day == 0:
          self.fp_days.add_component(Spacer(width=36))
        else:
          b = Button(text=str(day), role='cal-day')
          b.tag.date = datetime.date(year, month, day)
          b.set_event_handler('click', self.day_click)
          self.fp_days.add_component(b)

  def day_click(self, sender, **event_args):
    self.rp_day_detail.items = anvil.server.call('get_doses_for', sender.tag.date)
```

The "31/31 times this month" card is one server call counting `doses` rows where `taken=True` against the total for the month.

## 7. Notifications

The reminders in mockup 3 need to run without the app open, so use a **Scheduled Task** (Background Tasks → Scheduled Tasks, set to run daily). It scans yesterday's doses and low quantities and writes rows into `notifications`:

```python
@anvil.server.background_task
def nightly_check():
  yesterday = datetime.date.today() - datetime.timedelta(days=1)
  for dose in app_tables.doses.search(date=yesterday, taken=False):
    med = dose['medication']
    app_tables.notifications.add_row(
      user=med['user'], kind='missed',
      text=f"You forgot to take your {med['name']}!",
      created=datetime.datetime.now())
```

The Notifications form is then just a `RepeatingPanel` grouped by date, with a Role for the red "missed" styling versus the navy default.

## One thing worth flagging

Every server function should filter by `anvil.users.get_user()` and you should set the `medications` table permissions to **"No access"** from client code (forms) and do all reads/writes through server functions. Otherwise anyone with the app URL can query everyone's medication list from the browser console. For a school project it's also a nice thing to be able to point at in your writeup.
