# Offer Schema v0.1

**Version**: 0.1
**Status**: Draft
**Last Updated**: 2026-03-25

## Introduction

This document defines the protocol-facing `offer` object for AgentOffer Protocol v0.1.

The object is centered on:

- identity
- offer information (including category and commercial context)
- entity
- action
- targeting
- commission

### Conformance Keywords

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHOULD", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://datatracker.ietf.org/doc/html/rfc2119).

### Field Requirement Levels

The protocol classifies fields into three requirement levels to balance standardization with flexibility:

| Level | Label | Meaning |
|-------|-------|---------|
| **REQUIRED** | Required | Field MUST be present with a valid, non-empty value. |
| **RECOMMENDED** | Recommended | Field SHOULD be present and MUST follow the standard structure when present, but the value MAY be empty or null. |
| **OPTIONAL** | Optional | Field MAY be omitted entirely. When included, it SHOULD follow the specified format. |

## Offer

### Top-Level Shape

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `uuid` | string | REQUIRED | Stable offer identifier. UUIDv7 is recommended. |
| `version` | string | REQUIRED | Offer document version. Indicates which schema revision the offer instance conforms to. Current value: `"1.0"`. |
| `offer_info` | object | REQUIRED | Core descriptive, categorical, and commercial information for the offer. |
| `entity` | object | REQUIRED | Business entity the offer belongs to. |
| `action` | object | REQUIRED | Primary executable action exposed by the offer. |
| `material` | array | RECOMMENDED | Creative assets associated with the offer. |
| `targeting` | array | OPTIONAL | Targeting constraints that influence where the offer should be shown. |
| `commission` | object | OPTIONAL | Commission information associated with the offer. |
| `conversion_rule` | object | OPTIONAL | Rules for valid conversion events and attribution logic. |
| `frequency_capping` | object | OPTIONAL | Exposure frequency limits. |
| `tags` | array | OPTIONAL | Custom tags for filtering and semantic matching. |

## Object Model

### Identity

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `uuid` | string | REQUIRED | Stable offer identifier in the protocol response. UUIDv7 is recommended. |
| `version` | string | REQUIRED | Offer document version. Indicates which schema revision the offer instance conforms to. Current value: `"1.0"`. |

Design notes:

- `uuid` is the protocol-facing offer identity for query, tracking, and downstream reconciliation use cases.
- `version` enables forward compatibility when the schema evolves.

### Offer Information

`offer_info` groups the descriptive, categorical, and commercial information that belongs to the offer itself.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `offer_info.title` | string | REQUIRED | Display-ready title. |
| `offer_info.offer_type` | string | REQUIRED | Offer classification such as `physical_product`, `content`, `online_service`, or `offline_service`. |
| `offer_info.category` | object | REQUIRED | Industry category, vertical-specific attributes, and commercial details. |
| `offer_info.description` | string | REQUIRED | Core semantic description used by agents and clients. |
| `offer_info.source_offer_id` | string | OPTIONAL | Upstream source-side offer or inventory identifier. |
| `offer_info.start_at` | string | OPTIONAL | RFC 3339 timestamp indicating when the offer becomes active. |
| `offer_info.expire_at` | string | OPTIONAL | RFC 3339 timestamp indicating when the offer should no longer be surfaced. |
| `offer_info.status` | string | OPTIONAL | Lifecycle status such as `active`, `paused`, `pending`, `rejected`, or `expired`. |
| `offer_info.audit_status` | string | OPTIONAL | Compliance audit status such as `waiting`, `pass`, or `reject`. |
| `offer_info.priority` | integer | OPTIONAL | Exposure priority. Higher values take precedence in multi-offer competition. |

#### `offer_info.category`

`category` unifies the industry vertical classification, vertical-specific attributes, and commercial context into a single object.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `offer_info.category.type` | string | REQUIRED | Industry vertical. Registered values: `software_saas`, `travel_hospitality`, `education`, `financial_service`, `electronics`, `entertainment`. See **Category Types** section. |
| `offer_info.category.attributes` | object | RECOMMENDED | Vertical-specific attributes. Structure varies by `category.type`. See `category-attributes.types.ts` for per-type definitions. |
| `offer_info.category.commercial` | object | RECOMMENDED | Pricing, availability, and inventory information. |

##### `offer_info.category.commercial`

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `commercial.price.amount` | string | RECOMMENDED | Decimal string representing the consumer-facing price amount. |
| `commercial.price.currency` | string | RECOMMENDED | ISO 4217 currency code for the consumer-facing price. |
| `commercial.availability` | string | RECOMMENDED | Availability status such as `available`, `limited`, `sold_out`, or `pre_order`. |
| `commercial.stock` | integer | OPTIONAL | Available inventory count. Omit for unlimited inventory. |

Design notes:

- `offer_type` and `category.type` serve different purposes: `offer_type` classifies the **delivery form** (how the offer is fulfilled — physical product, online service, offline service, content), while `category.type` classifies the **industry vertical** (what domain the offer belongs to). An `offline_service` offer type could have a `category.type` of `travel_hospitality` or `education`.
- `category` keeps industry classification, vertical-specific data, and commercial context together. This reflects the natural relationship: commercial terms (pricing, stock) are inherently tied to the vertical category, not to the abstract offer envelope.
- `category.attributes` structure is determined by `category.type`. Each vertical defines its own required and optional attribute fields. For `entertainment`, an additional `sub_type` discriminator is used for finer classification.
- `commercial` is RECOMMENDED because some offers are action-oriented or promotional and may not expose pricing.

### Category Types

Six registered industry verticals. All types use `attributes.sub_type` for finer industry-specific classification. Each `sub_type` defines its own set of required and optional attribute fields.

| `category.type` | Description | `sub_type` values |
|-----------------|-------------|-------------------|
| `software_saas` | SaaS, subscription software, developer tools | `project_management`, `design`, `development_tools`, `crm`, `analytics`, `communication`, `security`, `ai_tools` |
| `travel_hospitality` | Hotels, flights, vacation rentals, dining | `hotel`, `flight`, `car_rental`, `vacation_package`, `restaurant`, `attraction` |
| `education` | Online courses, certification, training | `online_course`, `certification`, `bootcamp`, `language_learning`, `tutoring`, `academic_program` |
| `financial_service` | Credit cards, loans, insurance, payments | `credit_card`, `insurance`, `loan`, `investment`, `banking`, `payment` |
| `electronics` | Consumer electronics, smart devices | `smartphone`, `laptop`, `audio`, `wearable`, `gaming_hardware`, `smart_home`, `camera` |
| `entertainment` | Games, streaming, AI companions, betting | `game`, `streaming_video`, `ai_companion`, `social_audio`, `sports_betting`, `music_audio`, `live_streaming` |

Each type shares a set of common fields across all its sub_types, plus sub_type-specific fields. The tables below are the normative reference. See also `category-attributes.types.ts` for the machine-readable TypeScript definitions.

#### `software_saas` Attributes

Common fields shared across all `software_saas` sub_types:

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `sub_type` | string | REQUIRED | Software sub-category: `project_management`, `design`, `development_tools`, `crm`, `analytics`, `communication`, `security`, `ai_tools`. |
| `plan_type` | string | REQUIRED | Subscription or pricing model: `free_trial`, `freemium`, `paid`, `open_source`. |
| `platform` | array | REQUIRED | Supported platforms: `web`, `desktop`, `mobile`, `api`. |
| `trial_days` | integer | OPTIONAL | Free trial duration in days. |
| `features` | array | OPTIONAL | Key feature highlights. |
| `integrations` | array | OPTIONAL | Third-party integrations supported. |
| `seats_included` | integer | OPTIONAL | Number of seats included in the plan. |
| `deployment` | string | OPTIONAL | Deployment model: `cloud`, `on_premise`, `hybrid`. |

##### `software_saas` → `project_management`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `methodologies` | array | OPTIONAL | Supported methodologies: `kanban`, `scrum`, `waterfall`, `gantt`. |
| `max_projects` | integer | OPTIONAL | Maximum projects or boards allowed. |
| `time_tracking` | boolean | OPTIONAL | Whether time tracking is included. |

##### `software_saas` → `design`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `design_type` | string | OPTIONAL | Design tool category: `ui_ux`, `graphic`, `video`, `3d`, `whiteboard`. |
| `export_formats` | array | OPTIONAL | Export formats supported. |
| `real_time_collab` | boolean | OPTIONAL | Whether real-time collaboration is supported. |

##### `software_saas` → `development_tools`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `dev_category` | string | OPTIONAL | Tool category: `ide`, `ci_cd`, `hosting`, `monitoring`, `database`, `testing`. |
| `supported_languages` | array | OPTIONAL | Supported programming languages. |
| `self_hosted` | boolean | OPTIONAL | Whether self-hosted option exists. |

##### `software_saas` → `crm`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `max_contacts` | integer | OPTIONAL | Maximum contacts or leads allowed. |
| `email_automation` | boolean | OPTIONAL | Whether email automation is included. |
| `sales_pipeline` | boolean | OPTIONAL | Whether sales pipeline is included. |

##### `software_saas` → `analytics`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `analytics_type` | string | OPTIONAL | Analytics focus: `web`, `product`, `marketing`, `business_intelligence`. |
| `data_retention` | string | OPTIONAL | Data retention period. |
| `real_time` | boolean | OPTIONAL | Whether real-time dashboards are available. |

##### `software_saas` → `communication`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `channels` | array | OPTIONAL | Communication channels: `chat`, `video`, `voice`, `email`, `forum`. |
| `max_participants` | integer | OPTIONAL | Maximum participants in a call or meeting. |
| `screen_sharing` | boolean | OPTIONAL | Whether screen sharing is supported. |

##### `software_saas` → `security`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `security_type` | string | OPTIONAL | Security focus: `endpoint`, `network`, `identity`, `encryption`, `compliance`. |
| `certifications` | array | OPTIONAL | Compliance certifications held. |
| `soc2_compliant` | boolean | OPTIONAL | Whether SOC 2 compliant. |

##### `software_saas` → `ai_tools`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `ai_category` | string | OPTIONAL | AI capability: `text_generation`, `image_generation`, `code_assistant`, `data_analysis`, `automation`. |
| `model_provider` | string | OPTIONAL | Underlying model or engine. |
| `rate_limits` | string | OPTIONAL | API rate limits description. |

#### `travel_hospitality` Attributes

Common fields shared across all `travel_hospitality` sub_types:

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `sub_type` | string | REQUIRED | Travel sub-category: `hotel`, `flight`, `car_rental`, `vacation_package`, `restaurant`, `attraction`. |
| `destination` | object | REQUIRED | Destination location with `city` and `country` fields. |
| `cancellation_policy` | string | OPTIONAL | Cancellation policy summary. |

##### `travel_hospitality` → `hotel`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `property_type` | string | OPTIONAL | Property classification: `hotel`, `resort`, `hostel`, `apartment`, `villa`, `homestay`. |
| `star_rating` | number | OPTIONAL | Star or quality rating (1–5). |
| `amenities` | array | OPTIONAL | Available amenities. |
| `room_type` | string | OPTIONAL | Room or unit type. |
| `check_in_time` | string | OPTIONAL | Check-in time. |
| `check_out_time` | string | OPTIONAL | Check-out time. |
| `breakfast_included` | boolean | OPTIONAL | Whether breakfast is included. |
| `max_guests` | integer | OPTIONAL | Maximum guest count. |

##### `travel_hospitality` → `flight`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `airline` | string | REQUIRED | Airline name. |
| `route` | object | REQUIRED | Route with `origin` and `destination` fields. |
| `cabin_class` | string | REQUIRED | Cabin class: `economy`, `premium_economy`, `business`, `first`. |
| `stops` | integer | OPTIONAL | Number of stops (0 = direct). |
| `baggage_included` | boolean | OPTIONAL | Whether baggage is included. |
| `in_flight_wifi` | boolean | OPTIONAL | Whether in-flight wifi is available. |

##### `travel_hospitality` → `car_rental`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `vehicle_type` | string | REQUIRED | Vehicle category: `economy`, `compact`, `midsize`, `suv`, `luxury`, `van`. |
| `pickup_location` | string | REQUIRED | Pickup location. |
| `insurance_included` | boolean | OPTIONAL | Whether insurance is included. |
| `mileage_policy` | string | OPTIONAL | Mileage policy: `unlimited`, `limited`. |
| `driver_age_min` | integer | OPTIONAL | Minimum driver age. |

##### `travel_hospitality` → `vacation_package`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `duration_nights` | integer | REQUIRED | Duration in nights. |
| `includes` | array | REQUIRED | Included components: `flight`, `hotel`, `meals`, `activities`, `transfer`. |
| `group_size` | integer | OPTIONAL | Maximum group size. |
| `all_inclusive` | boolean | OPTIONAL | Whether the package is all-inclusive. |

##### `travel_hospitality` → `restaurant`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `cuisine_type` | string | REQUIRED | Cuisine type. |
| `price_range` | string | REQUIRED | Price range indicator: `$`, `$$`, `$$$`, `$$$$`. |
| `reservation_required` | boolean | OPTIONAL | Whether reservation is required. |
| `michelin_stars` | integer | OPTIONAL | Michelin stars (0–3). |
| `dietary_options` | array | OPTIONAL | Dietary options available. |

##### `travel_hospitality` → `attraction`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `attraction_type` | string | REQUIRED | Attraction classification: `theme_park`, `museum`, `tour`, `show`, `sports_event`. |
| `duration` | string | OPTIONAL | Duration description. |
| `age_restriction` | string | OPTIONAL | Age restriction. |
| `indoor_outdoor` | string | OPTIONAL | Setting: `indoor`, `outdoor`, `both`. |

#### `education` Attributes

Common fields shared across all `education` sub_types:

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `sub_type` | string | REQUIRED | Education sub-category: `online_course`, `certification`, `bootcamp`, `language_learning`, `tutoring`, `academic_program`. |
| `subject` | string | REQUIRED | Subject or topic area. |
| `level` | string | REQUIRED | Target learner level: `beginner`, `intermediate`, `advanced`, `professional`. |
| `language` | string | OPTIONAL | Language of instruction. |
| `certification` | boolean | OPTIONAL | Whether a certificate is awarded. |

##### `education` → `online_course`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `format` | string | REQUIRED | Delivery format: `video`, `interactive`, `live`, `self_paced`. |
| `duration_hours` | number | OPTIONAL | Course duration in hours. |
| `instructor` | string | OPTIONAL | Instructor name. |
| `enrollment_deadline` | string | OPTIONAL | Enrollment deadline. |

##### `education` → `certification`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `issuing_body` | string | REQUIRED | Issuing body or organization. |
| `validity_years` | integer | OPTIONAL | Certification validity in years. |
| `exam_format` | string | OPTIONAL | Exam format: `online`, `in_person`, `proctored`. |
| `prerequisites` | array | OPTIONAL | Prerequisites description. |

##### `education` → `bootcamp`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `format` | string | REQUIRED | Delivery format: `online`, `in_person`, `hybrid`. |
| `duration_weeks` | integer | REQUIRED | Duration in weeks. |
| `tech_stack` | array | OPTIONAL | Technology stack covered. |
| `job_placement_rate` | number | OPTIONAL | Job placement rate percentage. |
| `career_services` | boolean | OPTIONAL | Whether career services are included. |

##### `education` → `language_learning`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `target_language` | string | REQUIRED | Target language to learn. |
| `format` | string | REQUIRED | Learning format: `app`, `live_tutor`, `self_paced`, `immersive`. |
| `native_language_support` | array | OPTIONAL | Native languages supported for instruction. |
| `speech_recognition` | boolean | OPTIONAL | Whether speech recognition is used. |

##### `education` → `tutoring`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `format` | string | REQUIRED | Delivery format: `online`, `in_person`. |
| `session_duration_minutes` | integer | OPTIONAL | Session duration in minutes. |
| `tutor_qualification` | string | OPTIONAL | Tutor qualification description. |
| `group_size` | integer | OPTIONAL | Maximum group size (1 = private). |

##### `education` → `academic_program`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `institution` | string | REQUIRED | Institution name. |
| `degree_type` | string | REQUIRED | Degree type: `bachelor`, `master`, `phd`, `diploma`, `associate`. |
| `field_of_study` | string | REQUIRED | Field of study. |
| `duration_years` | number | OPTIONAL | Duration in years. |
| `format` | string | OPTIONAL | Delivery format: `on_campus`, `online`, `hybrid`. |
| `accreditation` | string | OPTIONAL | Accreditation body. |

#### `financial_service` Attributes

Common fields shared across all `financial_service` sub_types:

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `sub_type` | string | REQUIRED | Financial sub-category: `credit_card`, `insurance`, `loan`, `investment`, `banking`, `payment`. |
| `provider_license` | string | REQUIRED | Regulatory license or registration identifier. |

##### `financial_service` → `credit_card`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `annual_fee` | string | REQUIRED | Annual fee amount or description. |
| `apr_range` | string | REQUIRED | Annual percentage rate range. |
| `rewards_type` | string | REQUIRED | Rewards program type: `points`, `cashback`, `miles`, `none`. |
| `sign_up_bonus` | string | OPTIONAL | Sign-up bonus description. |
| `min_credit_score` | integer | OPTIONAL | Minimum credit score required. |
| `foreign_transaction_fee` | string | OPTIONAL | Foreign transaction fee percentage. |

##### `financial_service` → `insurance`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `insurance_type` | string | REQUIRED | Insurance product type: `life`, `health`, `auto`, `home`, `travel`, `pet`. |
| `premium_frequency` | string | REQUIRED | Premium payment frequency: `monthly`, `quarterly`, `annually`. |
| `coverage_amount` | string | OPTIONAL | Coverage amount description. |
| `deductible` | string | OPTIONAL | Deductible amount description. |
| `coverage_details` | array | OPTIONAL | Coverage details summary. |

##### `financial_service` → `loan`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `loan_type` | string | REQUIRED | Loan product type: `personal`, `mortgage`, `auto`, `student`, `business`. |
| `interest_rate_range` | string | REQUIRED | Interest rate range. |
| `term_months` | integer | REQUIRED | Loan term in months. |
| `max_amount` | string | OPTIONAL | Maximum loan amount. |
| `collateral_required` | boolean | OPTIONAL | Whether collateral is required. |

##### `financial_service` → `investment`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `investment_type` | string | REQUIRED | Investment product type: `brokerage`, `robo_advisor`, `crypto`, `fund`. |
| `min_investment` | string | OPTIONAL | Minimum investment amount. |
| `management_fee` | string | OPTIONAL | Management fee percentage. |
| `asset_classes` | array | OPTIONAL | Asset classes available. |

##### `financial_service` → `banking`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `account_type` | string | REQUIRED | Account type: `checking`, `savings`, `cd`, `money_market`. |
| `monthly_fee` | string | OPTIONAL | Monthly maintenance fee. |
| `apy` | string | OPTIONAL | Annual percentage yield. |
| `min_balance` | string | OPTIONAL | Minimum balance requirement. |
| `deposit_insured` | boolean | OPTIONAL | Whether FDIC or equivalent insured. |

##### `financial_service` → `payment`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `payment_type` | string | REQUIRED | Payment service type: `wallet`, `processor`, `transfer`, `bnpl`. |
| `supported_currencies` | array | REQUIRED | Supported currencies. |
| `transaction_fee` | string | OPTIONAL | Transaction fee description. |
| `settlement_time` | string | OPTIONAL | Settlement time description. |

#### `electronics` Attributes

Common fields shared across all `electronics` sub_types:

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `sub_type` | string | REQUIRED | Electronics sub-category: `smartphone`, `laptop`, `audio`, `wearable`, `gaming_hardware`, `smart_home`, `camera`. |
| `brand` | string | REQUIRED | Manufacturer brand. |
| `model` | string | REQUIRED | Product model name or number. |
| `condition` | string | REQUIRED | Item condition: `new`, `refurbished`, `used`. |
| `warranty_months` | integer | OPTIONAL | Warranty duration in months. |
| `color` | string | OPTIONAL | Color or finish. |

##### `electronics` → `smartphone`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `storage_gb` | integer | REQUIRED | Storage capacity in GB. |
| `screen_size_inches` | number | OPTIONAL | Screen size in inches. |
| `connectivity` | string | OPTIONAL | Connectivity standard: `5g`, `4g`. |
| `os` | string | OPTIONAL | Operating system: `ios`, `android`, `other`. |

##### `electronics` → `laptop`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `cpu` | string | REQUIRED | CPU model or description. |
| `ram_gb` | integer | REQUIRED | RAM in GB. |
| `storage_gb` | integer | OPTIONAL | Storage capacity in GB. |
| `screen_size_inches` | number | OPTIONAL | Screen size in inches. |
| `gpu` | string | OPTIONAL | GPU model or description. |

##### `electronics` → `audio`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `audio_type` | string | REQUIRED | Audio device type: `earbuds`, `headphones`, `speaker`, `soundbar`. |
| `noise_cancellation` | boolean | OPTIONAL | Whether active noise cancellation is supported. |
| `wireless` | boolean | OPTIONAL | Whether wireless connectivity is supported. |
| `battery_hours` | number | OPTIONAL | Battery life in hours. |

##### `electronics` → `wearable`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `wearable_type` | string | REQUIRED | Wearable device type: `smartwatch`, `fitness_tracker`, `smart_ring`, `smart_glasses`. |
| `os_compatibility` | array | OPTIONAL | Compatible operating systems. |
| `battery_days` | number | OPTIONAL | Battery life in days. |
| `water_resistance` | string | OPTIONAL | Water resistance rating. |
| `health_sensors` | array | OPTIONAL | Health sensors available. |

##### `electronics` → `gaming_hardware`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `hardware_type` | string | REQUIRED | Hardware type: `console`, `handheld`, `controller`, `vr_headset`. |
| `platform_ecosystem` | string | OPTIONAL | Platform ecosystem. |
| `storage_gb` | integer | OPTIONAL | Storage capacity in GB. |

##### `electronics` → `smart_home`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `device_type` | string | REQUIRED | Smart home device type: `speaker`, `display`, `camera`, `thermostat`, `lock`, `light`. |
| `voice_assistant` | string | OPTIONAL | Voice assistant compatibility. |
| `connectivity_protocol` | array | OPTIONAL | Connectivity protocols: `wifi`, `zigbee`, `matter`, `bluetooth`, `zwave`. |
| `hub_required` | boolean | OPTIONAL | Whether a hub is required. |

##### `electronics` → `camera`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `camera_type` | string | REQUIRED | Camera type: `dslr`, `mirrorless`, `action`, `instant`, `drone`. |
| `sensor_size` | string | OPTIONAL | Sensor size description. |
| `megapixels` | number | OPTIONAL | Megapixel count. |
| `video_resolution` | string | OPTIONAL | Maximum video resolution. |

#### `entertainment` Attributes

All `entertainment` offers MUST include a `sub_type` field that determines the sub-category. The following fields are shared across all entertainment sub-types:

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `sub_type` | string | REQUIRED | Entertainment sub-category discriminator. |
| `supported_devices` | array | OPTIONAL | Supported platforms or devices. |
| `age_rating` | string | OPTIONAL | Age restriction or content rating. |
| `languages` | array | OPTIONAL | Supported languages. |

##### `entertainment` → `game`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `platform` | array | REQUIRED | Supported gaming platforms: `pc`, `console`, `mobile`, `web`, `vr`. |
| `genre` | string | REQUIRED | Game genre. |
| `multiplayer` | boolean | OPTIONAL | Whether multiplayer is supported. |
| `developer` | string | OPTIONAL | Developer studio name. |
| `release_date` | string | OPTIONAL | Release date. |
| `in_app_purchases` | boolean | OPTIONAL | Whether in-app purchases exist. |
| `system_requirements` | object | OPTIONAL | Minimum system requirements as key-value pairs. |

##### `entertainment` → `streaming_video`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `content_type` | string | REQUIRED | Primary content type: `movie`, `series`, `short_drama`, `documentary`, `anime`, `live`, `variety`. |
| `subscription_model` | string | REQUIRED | Access model: `free`, `ad_supported`, `subscription`, `pay_per_view`. |
| `genres` | array | OPTIONAL | Content genres available. |
| `max_resolution` | string | OPTIONAL | Maximum streaming resolution. |
| `simultaneous_streams` | integer | OPTIONAL | Simultaneous stream count. |
| `offline_download` | boolean | OPTIONAL | Whether offline download is supported. |
| `original_content` | boolean | OPTIONAL | Whether platform produces original content. |

##### `entertainment` → `ai_companion`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `interaction_mode` | string | REQUIRED | Interaction modality: `text`, `voice`, `multimodal`. |
| `persona_type` | string | REQUIRED | Companion persona classification: `assistant`, `character`, `therapist`, `tutor`, `companion`, `roleplay`. |
| `customizable_persona` | boolean | OPTIONAL | Whether the user can customize the persona. |
| `memory_enabled` | boolean | OPTIONAL | Whether conversation memory is supported. |
| `content_filter` | string | OPTIONAL | Content safety filtering level: `strict`, `moderate`, `minimal`. |
| `voice_options` | array | OPTIONAL | Available voice options or TTS models. |

##### `entertainment` → `social_audio`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `interaction_type` | string | REQUIRED | Primary interaction type: `voice_chat`, `audio_room`, `podcast_live`, `karaoke`, `voice_match`. |
| `max_participants` | integer | OPTIONAL | Maximum participants per session. |
| `recording_enabled` | boolean | OPTIONAL | Whether session recording is available. |
| `moderation_tools` | boolean | OPTIONAL | Whether moderation tools are provided. |
| `monetization_support` | boolean | OPTIONAL | Whether creators can monetize. |

##### `entertainment` → `sports_betting`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `license_jurisdiction` | string | REQUIRED | Jurisdiction where the license is held. |
| `license_number` | string | REQUIRED | Regulatory license number. |
| `supported_sports` | array | REQUIRED | Sports categories available for betting. |
| `min_age` | integer | REQUIRED | Minimum legal age to participate. |
| `bet_types` | array | OPTIONAL | Supported bet types. |
| `live_betting` | boolean | OPTIONAL | Whether live / in-play betting is supported. |
| `cash_out` | boolean | OPTIONAL | Whether early cash-out is available. |
| `welcome_bonus` | string | OPTIONAL | Welcome bonus description. |
| `responsible_gambling_tools` | array | OPTIONAL | Responsible gambling tools provided. |

##### `entertainment` → `music_audio`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `content_type` | string | REQUIRED | Primary content type: `music`, `podcast`, `audiobook`, `radio`. |
| `subscription_model` | string | REQUIRED | Access model: `free`, `ad_supported`, `premium`. |
| `catalog_size` | string | OPTIONAL | Catalog size description. |
| `audio_quality` | string | OPTIONAL | Maximum audio quality. |
| `offline_mode` | boolean | OPTIONAL | Whether offline playback is supported. |
| `social_features` | boolean | OPTIONAL | Social or sharing features. |
| `lyrics_support` | boolean | OPTIONAL | Whether lyrics display is supported. |

##### `entertainment` → `live_streaming`

| Field | Type | Level | Description |
|-------|------|-------|-------------|
| `content_type` | string | REQUIRED | Primary content vertical: `gaming`, `entertainment`, `education`, `commerce`, `music`, `lifestyle`. |
| `interaction_model` | string | REQUIRED | Viewer interaction model: `watch`, `chat`, `gift`, `co_stream`. |
| `max_resolution` | string | OPTIONAL | Maximum stream resolution. |
| `monetization` | array | OPTIONAL | Monetization methods: `tips`, `subscriptions`, `ads`. |
| `low_latency` | boolean | OPTIONAL | Whether low-latency mode is available. |
| `vod_replay` | boolean | OPTIONAL | Whether VOD replay is available after live ends. |
| `streamer_tools` | array | OPTIONAL | Tools provided for streamers. |

### Entity

`entity` is the normalized business subject of the offer. It uses neutral wording so the model can work for merchants, brands, service providers, platforms, and other supply-side entities.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `entity.id` | string | REQUIRED | Stable entity identifier. |
| `entity.name` | string | REQUIRED | Display name of the entity. |
| `entity.type` | string | OPTIONAL | Entity classification such as `merchant`, `brand`, `creator`, `seller`, `service_provider`, or `financial_institution`. |
| `entity.description` | string | OPTIONAL | Short description of the entity. |
| `entity.website` | string | OPTIONAL | Canonical website for the entity. |

### Action

`action` describes how the offer is executed. `action.type` expresses the execution mechanism.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `action.type` | string | REQUIRED | Executable action type such as `web_redirect`, `app_deep_link`, or `api_trigger`. |
| `action.name` | string | RECOMMENDED | Short user-facing action name (CTA text). |
| `action.description` | string | OPTIONAL | Explanation of the action intent or destination. |
| `action.payload` | object | REQUIRED | Action-specific payload. Structure depends on `action.type`. |

#### `action.payload`

For a `web_redirect` action flow:

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `action.payload.target` | string | REQUIRED | Destination URL or executable target for the action. |
| `action.payload.requires_auth` | boolean | OPTIONAL | Whether the user must authenticate to complete the action. |
| `action.payload.platform` | string | OPTIONAL | Target platform such as `web`, `ios`, `android`, or `desktop`. |

### Material (RECOMMENDED)

`material` is an array of creative assets associated with the offer. It is RECOMMENDED: the field SHOULD be present, but when no assets are available the value MAY be an empty array.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `material[].image_url` | string | RECOMMENDED | URL to the creative asset. |
| `material[].tag` | string | RECOMMENDED | Asset purpose tag such as `logo`, `banner`, `hero`, or `thumbnail`. |
| `material[].format` | string | RECOMMENDED | Asset format such as `image`, `video`, or `html5`. |
| `material[].size` | string | OPTIONAL | Dimension specification such as `300x250`, `728x90`. |

### Targeting (OPTIONAL)

`targeting` is an optional list of constraints that can narrow where an offer should be surfaced.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `targeting[].geo.include` | array | OPTIONAL | Regions or markets explicitly included. |
| `targeting[].geo.exclude` | array | OPTIONAL | Regions or markets explicitly excluded. |
| `targeting[].language` | string | OPTIONAL | Preferred or required language context. |
| `targeting[].device_type` | array | OPTIONAL | Target device types such as `mobile`, `desktop`, `tablet`. |

### Commission (OPTIONAL)

`commission` expresses payout information for recommendation, ranking, or settlement workflows.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `commission.amount` | string | OPTIONAL | Decimal string representing the commission amount. |
| `commission.currency` | string | OPTIONAL | ISO 4217 currency code for the commission amount. |
| `commission.model` | string | OPTIONAL | Commission model such as `fixed`, `percentage`, `cpa`, or `cps`. |

### Conversion Rule (OPTIONAL)

`conversion_rule` defines what constitutes a valid conversion and how attribution is assigned.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `conversion_rule.trigger` | string | OPTIONAL | Event type that triggers a valid conversion (e.g., `purchase`, `booking_complete`, `signup`, `install`). |
| `conversion_rule.window_hours` | integer | OPTIONAL | Attribution window in hours from the initial click. |
| `conversion_rule.attribution_type` | string | OPTIONAL | Attribution model such as `last_click`, `first_click`, or `multi_touch`. |

### Frequency Capping (OPTIONAL)

`frequency_capping` prevents user overexposure to the same offer.

| Field | Type | Level | Description |
|------|------|-------|-------------|
| `frequency_capping.per_user_day` | integer | OPTIONAL | Maximum impressions per user per day. |
| `frequency_capping.per_user_total` | integer | OPTIONAL | Maximum lifetime impressions per user. |

### Tags (OPTIONAL)

`tags` is an optional array of strings for filtering, semantic matching, and audience targeting.

## Validation Rules

### REQUIRED Fields

- `uuid` MUST be a non-empty string. UUIDv7 format is recommended.
- `version` MUST be a non-empty string.
- `offer_info.title` MUST be a non-empty string suitable for direct display.
- `offer_info.offer_type` MUST be a non-empty string that classifies the offer.
- `offer_info.category.type` MUST be one of the registered category type values.
- `offer_info.description` MUST be a non-empty string suitable for semantic retrieval and end-user display.
- `entity.id` and `entity.name` are both REQUIRED.
- `action.type` and `action.payload` are both REQUIRED.
- `action.payload.target` MUST be a valid URI when the action is a web flow.

### RECOMMENDED Fields

These fields SHOULD be present in the offer object and MUST follow the standard structure when present. Values MAY be empty or null when data is unavailable, but the field key itself SHOULD exist:

- `material` SHOULD be present as an array. MAY be an empty array `[]` when no assets are available. When items are provided, each SHOULD include `image_url`, `tag`, and `format`.
- `category.attributes` SHOULD be present as an object. MAY be an empty object `{}` when no vertical-specific data is available.
- `category.commercial` SHOULD be present as an object. `price.amount` and `price.currency` MAY be empty strings when pricing is not exposed.

### OPTIONAL Fields

These fields MAY be omitted entirely. When included, they SHOULD follow the specified format:

- `offer_info.start_at` and `offer_info.expire_at`, when present, MUST be valid RFC 3339 timestamps.
- `entity.website`, when present, SHOULD be a valid URI.
- `conversion_rule`, when present, SHOULD include `trigger`, `window_hours`, and `attribution_type`.

## Example

```json
{
  "uuid": "0195ef94-f17d-7a4f-b6e0-2c52bb49e142",
  "version": "1.0",
  "offer_info": {
    "title": "The Manhattan Grand — Deluxe King Room",
    "offer_type": "offline_service",
    "category": {
      "type": "travel_hospitality",
      "attributes": {
        "property_type": "hotel",
        "destination": {
          "city": "New York",
          "country": "US"
        },
        "star_rating": 5,
        "amenities": ["Rooftop pool", "Spa & wellness center", "24-hour concierge"],
        "room_type": "Deluxe King",
        "breakfast_included": true,
        "cancellation_policy": "Free cancellation up to 48 hours before check-in"
      },
      "commercial": {
        "price": {
          "amount": "420.00",
          "currency": "USD"
        },
        "availability": "limited",
        "stock": 15
      }
    },
    "description": "Luxury 5-star hotel in Midtown Manhattan with rooftop pool, full-service spa, and complimentary breakfast.",
    "start_at": "2026-04-01T00:00:00Z",
    "expire_at": "2026-10-31T23:59:59Z",
    "status": "active",
    "audit_status": "pass",
    "priority": 9
  },
  "material": [
    {
      "image_url": "https://manhattangrand.example/deluxe_king_300x250.jpg",
      "tag": "banner",
      "format": "image",
      "size": "300x250"
    }
  ],
  "entity": {
    "id": "ent_manhattan_grand",
    "name": "The Manhattan Grand Hotel",
    "type": "service_provider",
    "website": "https://www.manhattangrand.example"
  },
  "action": {
    "type": "web_redirect",
    "name": "Book now",
    "description": "Redirect to the hotel booking page to reserve your room.",
    "payload": {
      "target": "https://www.manhattangrand.example/book/deluxe-king",
      "requires_auth": false,
      "platform": "web"
    }
  },
  "targeting": [
    {
      "geo": { "include": ["US", "GB", "CA"] },
      "language": "en",
      "device_type": ["mobile", "desktop", "tablet"]
    }
  ],
  "commission": {
    "amount": "42.00",
    "currency": "USD",
    "model": "cpa"
  }
}
```

## Example Coverage

The protocol examples cover all six registered category types:

| Category Type | Example File | Notes |
|--------------|-------------|-------|
| `software_saas` | `notion-offer.json` | Minimal example (REQUIRED + RECOMMENDED only) |
| `electronics` | `product-offer.json` | Full offer with all OPTIONAL fields |
| `education` | `content-offer.json` | Full offer |
| `travel_hospitality` | `offline-service-offer.json` | Full offer |
| `financial_service` | `financial-service-offer.json` | Full offer with regulatory attributes |
| `entertainment` | `entertainment-offer.json` | Full offer with `app_deep_link` action |

## Design Decisions

- **Requirement Levels (RFC 2119)**: REQUIRED enforces a strict core contract (must have valid values). RECOMMENDED enforces structural consistency across implementations (field should exist and follow the standard shape, even if the value is empty) — this ensures all offer documents share the same structure for parsing and tooling. OPTIONAL fields allow richer modeling without breaking backward compatibility.
- **`category` consolidation**: Industry type, vertical-specific attributes, and commercial terms are inherently related. Grouping them under `category` reduces top-level field sprawl and makes it clear that pricing and availability are category-dependent.
- **`material` as array**: A single offer may need multiple creative assets (logo, banner, video). The array structure handles this naturally.
- **OPTIONAL for targeting, commission, conversion, frequency, tags**: These are powerful features but not every offer needs them. Keeping them optional prevents the protocol from forcing complexity on simple use cases.

## Changelog

| Version | Date | Changes |
|---------|------|---------|
| 0.1 | 2026-03-20 | Initial draft. |
| 0.1 | 2026-03-22 | Added compatibility-oriented fields and industry mapping guidance. |
| 0.1 | 2026-03-23 | Added `primary_action` and `action_labels` for CTA semantics. |
| 0.1 | 2026-03-24 | Reframed the protocol around `offer`, `entity`, and `action`, simplified field guidance to required versus optional, and aligned key names around `id`, `offer_type`, `source_offer_id`, and user-facing action semantics. |
| 0.1 | 2026-03-24 | Proposed `uuid`, `offer_info`, executable `action.type`, and optional `targeting` plus `commission` as the next draft shape. |
| 0.1 | 2026-03-24 | Refined the draft with `offer_info.offer_type`, `offer_info.source_offer_id`, `commercial.availability`, `entity.website`, and `commission.model`. |
| 0.1 | 2026-03-25 | Introduced REQUIRED/RECOMMENDED/OPTIONAL requirement levels (RFC 2119). Unified `industry` + `industry_attributes` + `commercial` into `offer_info.category`. Renamed `creative` to `material` (array, RECOMMENDED). Reclassified `targeting`, `commission`, `conversion_rule`, `frequency_capping`, and `tags` as OPTIONAL. Added `version` field. Removed `tracking`. Defined 6 category types with entertainment sub_type system. |
