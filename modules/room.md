# Room Module

## Purpose
The Room module handles the complete **hotel room booking flow** — from browsing offers, selecting dates on a calendar, viewing available rooms with rates, to confirming and adding a room to the cart. It also manages **casino/loyalty player authentication** for properties that support loyalty programs.

## Location
`src/app/room/`

## Folder Structure
```
room/
├── room.module.ts                    # NgModule declaration
├── room-routing.module.ts            # Route definitions (lazy-loaded)
│
├── service/
│   ├── book-resolver-service.ts      # Route resolver — initializes app data before rendering
│   ├── pms-filter-service.ts         # PMS filter state management (signals-based)
│   └── wbe-room-service/
│       ├── wbe-room.service.ts       # HTTP service — raw API calls
│       ├── wbe-room-transformer.service.ts  # Business logic layer over HTTP
│       ├── wbe-cms-transformer-service.ts   # Casino/loyalty CMS logic
│       ├── wbe-room.constant.ts      # API endpoint URL templates
│       └── model/                    # TypeScript interfaces
│           ├── calendarDetails/      # Calendar API response models
│           ├── common/               # Shared models (Room, Wbe, Offers)
│           ├── enhancementDetails/   # Enhancement/add-on models
│           ├── loyaltyDetails/       # Casino/loyalty player models
│           ├── offerFilter/          # Offer filter & guest type models
│           ├── requestPayload/       # API request payloads
│           ├── roomDetails/          # Room rate response models
│           └── roomFilter/           # Room filter params
│
└── widgets/
    ├── landing/                      # RoomMainComponent — parent shell
    ├── calendar/                     # CalendarComponent — date selection
    │   └── calendar-actions/         # CalendarActionsComponent — proceed/back
    ├── rooms-rates/                  # RoomsRatesComponent — room listing + rates
    │   ├── advanced-room-filter/     # AdvancedRoomFilterComponent
    │   └── calendar-popup/           # CalendarPopUpComponent
    ├── loyalty/                      # LoyaltyComponent — casino/loyalty login
    ├── room-confirmation-dialog/     # RoomConfirmationDialogComponent — add-to-cart confirm
    ├── policymodel/                  # PolicymodelComponent — cancellation/policy popup
    ├── resolver/                     # RoomResolverService — route data resolver
    ├── can-activate/                 # Route guards
    │   ├── check-products-access.service.ts
    │   ├── loyalty-resolver.service.ts
    │   └── room-routing-permission-resolver.service.ts
    └── router-guard/
        └── can-deactivate.guard.ts
```

---

## Architecture Pattern

Every widget follows a **Component → TransformerService → ViewModel** pattern:

```
┌──────────────────┐     ┌──────────────────────────┐     ┌──────────────┐
│   Component      │────▶│   TransformerService      │────▶│   ViewModel  │
│ (calendar.ts)    │     │ (calendar-transformer.ts) │     │ (calendar-   │
│                  │◀────│                           │     │  viewmodel)  │
│ Handles UI events│     │ Business logic, API calls │     │ Data binding │
└──────────────────┘     └──────────────────────────┘     └──────────────┘
                                     │
                                     ▼
                         ┌──────────────────────────┐
                         │ WbeRoomTransformerService │
                         │ (URL building + HTTP)     │
                         └──────────────────────────┘
                                     │
                                     ▼
                         ┌──────────────────────────┐
                         │ WbeRoomService            │
                         │ (Raw HttpClient calls)    │
                         └──────────────────────────┘
```

---

## Routing

All routes are **children of `RoomMainComponent`** (the shell). Defined in `room-routing.module.ts`:

| Route Pattern | Component | Purpose |
|---|---|---|
| `offers/:tenant/:property` | `OffersComponent` (shared) | Browse available offers |
| `offers/:tenant/:property/:arrival/:departure` | `OffersComponent` | Offers with pre-selected dates |
| `loyalty/:tenant/:property/:guid` | `LoyaltyComponent` | Casino/loyalty player login via GUID |
| `calendar/:tenant/:property/:offer` | `CalendarComponent` | Select dates for an offer |
| `calendar/:tenant/:property/:offer/:guest` | `CalendarComponent` | Calendar with guest count |
| `calendar/:tenant/:property/:arrival/:departure/:offer/:guest` | `CalendarComponent` | Calendar with pre-filled dates |
| `room/:tenant/:property/:arrival/:departure/:offer/:guest` | `RoomsRatesComponent` | View rooms & rates |
| `room/:tenant/:property/:arrival/:departure` | `RoomsRatesComponent` | Rooms without offer code |
| `session-timeout/:tenant/:property` | `SessionTimeoutComponent` | Session expired page |

**Route Guards:**
- `RoomRoutingPermisssionResolverService` — validates user has permission to access room routes
- `LoyaltyResolverService` — validates GUID and loyalty data before loyalty page loads

---

## Key Components — Detailed Logic

### 1. `RoomMainComponent` (Landing / Shell)
**File:** `widgets/landing/room.component.ts`

- Parent wrapper component with a `<router-outlet>` for child routes
- Subscribes to `ActivatedRoute.data` to receive resolved PMS data
- Holds `propertyInfoResponse` as an Angular **signal**
- Minimal logic — acts purely as a layout shell

---

### 2. `CalendarComponent` (Date Selection)
**File:** `widgets/calendar/calendar.component.ts`

**Purpose:** Displays an interactive calendar for the guest to pick arrival/departure dates based on offer availability.

**Initialization Flow:**
```
1. Constructor:
   ├─ Get product data (Wbe) from WbePropertyTransformerService
   ├─ Get rGuestMessages (i18n labels)
   ├─ Initialize CalendarViewModel via CalendarTransformerService
   ├─ Subscribe to ActivatedRoute.parent.data → get PMS response
   ├─ Subscribe to route params → extract offer code
   ├─ Merge params + queryParams → set guest count data
   ├─ Open CalendarActionsComponent (proceed/back buttons)
   └─ Set calendar weeks from PMS response date config

2. Effects (Angular Signals):
   ├─ Watch internationalization changes → update calendar labels
   ├─ Watch loyalty GUID data → handle combined guest login flow
   └─ Watch calendar restrictions (shoulder dates, sold out, etc.)
```

**Key Logic:**
- **Multi-year calendar**: Supports year dropdown navigation for long-lead bookings
- **Date restrictions**: Handles closed-to-arrival, closed-to-departure, sold-out, minimum length of stay, and group shoulder dates
- **Combined flow**: When `guestLoginFlow === 'Combined'`, loyalty and calendar are merged
- Delegates all business logic to `CalendarTransformerService`

**CalendarTransformerService** (`calendar-transformer.service.ts`):
- Calls `WbeRoomTransformerService.calendarOfferDetails()` to fetch calendar availability
- Calls `calendarOfferRestrictions()` for date restriction rules
- Processes calendar data into week-based grid (`ModifiedCalendarData`, `WeekData`)
- Manages offer code parsing, guest count state, snackbar notifications for restrictions
- Uses signals for reactive state: `offerCodeSearched`, `selectedOffer`, `calendarSnackbarAction`

---

### 3. `RoomsRatesComponent` (Room Listing & Rates)
**File:** `widgets/rooms-rates/room-rates.component.ts`

**Purpose:** Displays available rooms with prices, images, amenities, and "Book" buttons for the selected dates.

**Key Features:**
- **Room list rendering** with sort/filter capabilities
- **Advanced room filter** (bed type, amenities, price range) via `AdvancedRoomFilterComponent`
- **Room detail popup** via `RoomDetailComponent` (shared widget)
- **Policy modal** via `PolicymodelComponent` for cancellation/guarantee policies
- **Room confirmation dialog** when user clicks "Book"
- **360 view / video** templates for room media
- **Price calculation** with currency formatting
- **Calendar popup** for date changes without leaving the page

**RoomRatesTransformerService** (`room-rates-transformer.service.ts`):
- Calls `WbeRoomTransformerService.roomDetails()` to fetch room availability + pricing
- Manages cart data state for room selections
- Handles enhancement (add-on) fetching via `enhanceStayDetails()`
- Handles offer list from DB via `offerList()`
- Manages casino player details state for loyalty-priced rooms

---

### 4. `LoyaltyComponent` (Casino/Loyalty Login)
**File:** `widgets/loyalty/loyalty.component.ts`

**Purpose:** Handles casino/loyalty player authentication. Guests enter their casino number + PIN to get loyalty-tier pricing.

**Flow:**
```
1. Extract GUID from route params
2. Subscribe to parent route data → get PMS response
3. Call loyaltyTransformerService.setGuidData() → validate GUID
4. Call setViewModelData() → initialize loyalty UI
5. Watch query params → handle route redirections
6. Effects:
   ├─ Language change → update labels
   ├─ Calendar group shoulder dates → toggle availability flags
   └─ Multi-year calendar → populate year dropdown
```

**CmsTransformerService** (`wbe-cms-transformer-service.ts`):
- Manages casino player login state via signals (`isPlayerLogin`, `isPlayerLogout`, `casinoPlayerloginCall`)
- Tracks primary + secondary casino player details
- Handles loyalty token lifecycle (refresh intervals)
- Supports multiple authentication types:
  - Direct casino login (casino number + PIN)
  - Gigya SSO login (third-party)
  - GUID-based login (link from email)

---

### 5. `RoomConfirmationDialogComponent`
**File:** `widgets/room-confirmation-dialog/room-confirmation-dialog.component.ts`

**Purpose:** Modal dialog shown when user selects a room — confirms room details, guest count, enhancements before adding to cart.

**Features:**
- Shows room images (with 360 view and video support)
- Guest count editor with `MatMenuTrigger`
- Enhancement selection
- Policy viewing
- "Add to Cart" action

---

### 6. `PolicymodelComponent`
**File:** `widgets/policymodel/policymodel.component.ts`

**Purpose:** Dialog showing cancellation policy, guarantee policy, and cancel-by dates for the selected rate.

---

## Services — Layered Architecture

### Layer 1: `WbeRoomService` (HTTP Layer)
**File:** `service/wbe-room-service/wbe-room.service.ts`

Pure HTTP methods — no business logic:

| Method | HTTP | Purpose |
|---|---|---|
| `getCalendarOfferDetails()` | GET | Calendar availability (v1) |
| `getCalendarOfferDetails_1()` | GET | Calendar availability (v2) |
| `postCalendarDetails()` | POST | Calendar details with payload |
| `getDateRestrictions()` | PUT | Date restriction rules |
| `getEnhanceDetails()` | GET | Enhancement/add-on options |
| `getRoomDetails()` | GET | Room rates + availability |
| `modifyRoomDetails()` | PUT | Modify existing reservation |
| `postEnhanceDetails()` | POST | Add enhancements to reservation |
| `validateCasinoLogin()` | PUT | Casino number + PIN validation |
| `getLoyaltyToken()` | GET | Get loyalty auth token |
| `getPlayerDetails()` | GET | PENN Gaming player details |
| `getCasinoPlayerDetails()` | GET | Casino player profile |
| `getGuidDetails()` | GET | Validate loyalty GUID link |
| `getCasinoFields()` | GET | Casino system config from admin |
| `getOfferList()` | GET | Offer list from DB |
| `getRoomLocaleList()` | GET | Room locale/language data |

### Layer 2: `WbeRoomTransformerService` (Business Logic Layer)
**File:** `service/wbe-room-service/wbe-room-transformer.service.ts`

- Builds URLs using `WbeRoomConstant` templates + `UrlBuilder.replaceParams()`
- Adds conditional query params (casino number, yield flags, end dates, nightly rates)
- Reads tenant/property config from `WbePropertyTransformerService`
- Also handles **villa/unit pricing** via `getVillaPriceDetails()` (shares some logic with unit module)

### Layer 3: `WbeRoomConstant` (API Endpoints)
**File:** `service/wbe-room-service/wbe-room.constant.ts`

URL templates with `##paramName` placeholders. Key backend services called:

| Backend Service | Endpoint Prefix | Purpose |
|---|---|---|
| `wbe-calendar-service` | `/wbe/calendar` | Calendar availability |
| `wbe-rate-service` | `/rate` | Room rates & pricing |
| `wbe-enhancestay-service` | `/enhancestay` | Enhancement add-ons |
| `wbe-loyalty-service` | `/loyalty` | Casino/loyalty auth |
| `wbe-cms-service` | `/wbe/cms` | Casino validation |
| `wbe-offer-service` | `/offer` | Offer listing |
| `wbe-updateres-service` | `/cico/updateres` | Reservation modifications |
| `wbe-property-service` | `/room` | Room locale data |

### `PmsfilterService` (State Management)
**File:** `service/pms-filter-service.ts`

Central signal-based state store for room module state:

| Signal | Type | Purpose |
|---|---|---|
| `offerCodeSearched` | `string \| OfferCodeSearchedParams` | Current offer code from search |
| `selectedOffer` | `string \| boolean` | Active selected offer |
| `selectedOfferFromRoom` | `string \| SelectedOfferPath` | Offer selected from room page |
| `routeRoomFilterData` | `string \| RoomFilterDataParams` | Filter params for room route |
| `calendarSnackbarAction` | `string` | Calendar restriction snackbar state |
| `roomFilterDate` | `string \| DateRangePickerInput` | Date range for room filter |
| `isCalendarGroupShoulderDate` | `boolean` | Shoulder date restriction flag |
| `isCalendarClosedToArrival` | `boolean` | Closed-to-arrival flag |
| `isCalendarClosedToDeparture` | `boolean` | Closed-to-departure flag |
| `isCalendarSoldOut` | `boolean` | Sold out flag |
| `isCalendarMinimumLengthOfStay` | `boolean` | Min stay restriction flag |
| `loyaltyGuidData` | via signal | Loyalty GUID for combined flow |

### `BookResolverService`
**File:** `service/book-resolver-service.ts`

Route resolver that initializes required data before any room page renders:
- Loads property config, casino system config
- Sets up guest type limits
- Manages transaction IDs
- Handles casino detail call orchestration

---

## User Flow — End to End

```
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│   Offers     │───▶│   Calendar   │───▶│  Rooms &     │───▶│  Confirmation│
│   Page       │    │   Page       │    │  Rates       │    │  Dialog      │
│              │    │              │    │              │    │              │
│ Browse offers│    │ Pick dates   │    │ View rooms   │    │ Review &     │
│ (shared)     │    │ Check avail. │    │ Compare rates│    │ Add to Cart  │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
       │                                                           │
       ▼                                                           ▼
┌──────────────┐                                          ┌──────────────┐
│  Loyalty     │                                          │   Cart       │
│  Login       │                                          │   Module     │
│  (optional)  │                                          │              │
│ Casino #/PIN │                                          │ Checkout...  │
└──────────────┘                                          └──────────────┘
```

### Detailed Step-by-Step:

1. **Guest arrives at Offers page** → browses available rate offers
2. **Selects an offer** → navigates to `calendar/:tenant/:property/:offer`
3. **Calendar page loads:**
   - Fetches calendar availability from `wbe-calendar-service`
   - Fetches date restrictions
   - Guest picks arrival + departure dates
   - Guest sets guest count (adults, children, etc.)
4. **Proceeds to Rooms & Rates** → navigates to `room/:tenant/:property/:arrival/:departure/:offer/:guest`
5. **Room listing page loads:**
   - Fetches room rates from `wbe-rate-service`
   - Displays rooms with images, amenities, pricing
   - Guest can sort/filter rooms
6. **Guest clicks "Book"** on a room → `RoomConfirmationDialogComponent` opens
   - Shows room details, allows enhancement selection
   - Shows cancellation policy (via `PolicymodelComponent`)
7. **Guest confirms** → room added to cart via `CommonCartLayer` / `CartService`
8. **Cart module** takes over for checkout

### Loyalty/Casino Flow (Optional):
- If property has casino integration, guest may land on `loyalty/:tenant/:property/:guid`
- `CmsTransformerService` validates casino login
- Loyalty token is obtained and passed to subsequent API calls
- Loyalty-tier pricing is reflected in room rates

---

## Data Models

### Key Interfaces

| Model | Location | Purpose |
|---|---|---|
| `Wbe` | `model/common/wbe.ts` | Main product configuration (PMS type, permissions, property keys, rGuestMessages) |
| `Room` / `Room2` | `model/common/room.ts` | Room data (code, name, amenities, images) |
| `GetRoomsResponse` | `model/roomDetails/get-rooms-response.ts` | API response for room listing |
| `GetCalendarDetailsUpdatedResponse` | `model/calendarDetails/` | Calendar availability grid |
| `ModifiedCalendarData` / `WeekData` | `model/calendarDetails/modified-calendar-data.ts` | Processed calendar for UI display |
| `DateRestrictionResponse` | `model/calendarDetails/date-restrictions-response.ts` | Date restriction rules |
| `EnhanceDataResponse` | `model/enhancementDetails/enhance-data-response.ts` | Enhancement/add-on options |
| `CasinoPlayerDetailsResponse` | `model/loyaltyDetails/casino-player-details-response.ts` | Casino player profile |
| `CasinoTokenResponse` | `model/loyaltyDetails/casino-token-response.ts` | Casino auth token |
| `LoyalTokenResponse` | `model/loyaltyDetails/loyal-token-response.ts` | Loyalty auth token |
| `CalendarDetailsPayload` | `model/requestPayload/calendar-details-payload.ts` | Calendar API request body |
| `RoomConfirmationDialogData` | `model/roomDetails/room-confirmation-dialog-data.ts` | Data passed to confirmation modal |
| `OfferDetails` | `model/common/offer-details.ts` | Selected offer information |

---

## Dependencies on Shared Module

| Shared Service/Widget | Usage |
|---|---|
| `WbePropertyTransformerService` | Tenant/property config, `appViewModelData` |
| `SharedTransformerService` | Cross-module state (calendar year list, etc.) |
| `HeaderTransformerService` | Header state updates (guest name, login status) |
| `InternationalizationTransformerService` | Language change handling |
| `GoogleAnalyticsEventsService` | Event tracking |
| `WbePreloaderTransformerService` | Loading spinner control |
| `CommonCartLayer` | Add items to cart |
| `WbeOfferTransformerService` | Offer data shared across modules |
| `AppResolverService` | App-level initialization data |
| `OffersComponent` | Shared offers page (displayed as child route) |
| `EnhancementsComponent` | Enhancement selection dialog |
| `RoomDetailComponent` | Room detail popup |
| `SessionTimeoutComponent` | Session expired page |

---

## State Management Approach

The room module uses **Angular Signals** (not NgRx) for reactive state:

```
PmsfilterService          → Signals for filter state, offer codes, dates, restrictions
CmsTransformerService     → Signals for casino login state (isPlayerLogin, tokenExpired)
CalendarTransformerService → BehaviorSubjects + Signals for calendar data
RoomRatesTransformerService → Signal-based room response state
```

**Session Storage** (`ngx-webstorage`): Used for persisting language selection, loyalty tokens across page refreshes.

---

## Multi-Tenant / PMS Considerations

- The `Wbe.pms` field determines which PMS backend (e.g., `'stay'`) is active
- `Wbe.permissions.functionalPermissions` controls feature flags:
  - `multiGuestType` — allows children/infant counts
  - `allowEnhancementSelectionForSpecificDays` — nightly enhancement pricing
- `Wbe.permissions.routePermissions` controls which pages are accessible:
  - `enhancements` — whether enhancement step is shown
- `Wbe.guestLoginFlow` determines auth flow type:
  - `'Combined'` — loyalty + calendar on same page
  - Standard — separate loyalty page
- `Wbe.PropNameKeyList` holds property-specific keys for API calls (rateslist, roomdescription, enhance, history, validCasino)

---

## Common Tasks

### Add a new field to room listing
1. Update `GetRoomsResponse` model in `model/roomDetails/get-rooms-response.ts`
2. Update `RoomsRates` viewmodel in `room-rates-viewmodel.ts`
3. Map the field in `RoomRatesTransformerService`
4. Bind in `room-rates.component.html`

### Add a new calendar restriction type
1. Add signal in `PmsfilterService`
2. Handle in `CalendarTransformerService` restriction processing
3. Update `CalendarViewModel` for UI state
4. Show snackbar or visual indicator in `calendar.component.html`

### Debug casino/loyalty login
1. Check `CmsTransformerService.isPlayerLogin()` signal
2. Check `CmsTransformerService.loyaltyToken` value
3. Check network calls to `wbe-loyalty-service` and `wbe-cms-service`
4. Verify `Wbe.PropNameKeyList.validCasino` is configured

### Add a new route to room module
1. Add route entry in `room-routing.module.ts`
2. Create component in `widgets/`
3. Add transformer service + viewmodel
4. Add `canActivate` guard if access control is needed
5. Declare component in `room.module.ts`

---

## Gotchas & Notes

- **`wbe-room-transformer.service.ts` has commented-out code**: The `fetchRooms()` method is fully commented out (~50 lines). This was likely the old room fetching logic before refactoring. Do not remove without checking git history.
- **`Room` vs `Room2` models**: Two room models exist in `model/common/`. `Room2` appears to be an extended version. Check which is used where before modifying.
- **Casino player token refresh**: `CmsTransformerService` sets intervals (`primaryCasinoPlayerTokenInterval`, `secondaryCasinoPlayerTokenInterval`) to refresh loyalty tokens. These must be cleared on logout to avoid memory leaks.
- **Villa/unit pricing in room transformer**: `WbeRoomTransformerService.getVillaPriceDetails()` handles unit/villa pricing — this creates a cross-module dependency with the `unit` module.
- **`_1` suffix methods**: `calendarOfferDetails_1()` and `getCalendarOfferDetails_1()` are v2 API variants. The naming is legacy — treat them as the current version.
- **jQuery dependency**: `declare var $: any` appears in some components. This is used for DOM manipulation (likely legacy). Avoid adding more jQuery usage.
- **`console.log` statements**: Multiple `console.log` calls exist in production code (e.g., in `getLoyaltyToken()`). These are left intentionally for debugging but should be gated behind environment flags ideally.
