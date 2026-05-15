<!--
   Copyright 2026 UCP Authors
   Licensed under the Apache License, Version 2.0 (the "License");
   you may not use this file except in compliance with the License.
   You may obtain a copy of the License at
       http://www.apache.org/licenses/LICENSE-2.0
   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->

# Loyalty Extension

## Overview

The loyalty extension is designed to facilitate high-fidelity loyalty experiences: ensuring existing loyalty members can seamlessly access their benefits during agentic Cart and Checkout experiences. By enabling buyers to see their specific tier, eligible rewards, and immediately applicable benefits before finalizing a purchase, it addresses a foundational expectation for program members and removes friction from the checkout funnel.

Specifically the following core use cases of benefit recognition for known members are addressed:

* Price-Impacting Benefits: Real-time application of member-only discounts and free shipping offers with clear attribution of benefit sources, including multiplicative benefits unlocked by concurrent memberships.
* Non-Price Benefits: Transparent display of rewards earned or rewards applicable to future purchases.
* Status Recognition: Verification and display of the buyers’ specific loyalty tier within a program.

## Key Concepts

Loyalty has four main components:

**Memberships**: Distinct enrollment pathways or program categories that a user can join

* Independent programs offered by the same brand (e.g., a "Rewards Club" vs. a "Co-branded Credit Card") are modeled as separate, independently verifiable memberships. They are programmatically represented as separate sibling top-level keys in the loyalty extension map, namespaced by reverse-domain naming.

**Tiers**: Specific achievement ranks or status milestones within a membership that unlock escalating value as a member progresses through activity or spend

* A member typically holds a single active tier per membership. For programs with parallel status dimensions (e.g., holding both "Gold" and "Lifetime Platinum"), multiple tiers can be active concurrently.

**Benefits**: Ongoing perks and privileges granted to a customer based on their current tier or membership status

* Contains both delayed (e.g. “Members have access to dedicated customer service”) and immediate-value (e.g. “Members get 5% off”) benefits.

**Rewards**: Quantifiable loyalty value that may be earned from the current transaction. Note: Redeemable balances and stored value are modeled by the negotiated payment instrument or a future redemption capability, not by this loyalty extension.

* One membership can offer multiple types of accumulable/collectable rewards, each having its own usage and redemption rules.

```json
{
  "loyalty": {
    "com.example.loyalty" : {
      "tiers": [
        {
          "benefits": [
            {
              ...
            }
          ]
        }
      ],
      "rewards": [
        {
          ...
        }
      ]
    }
  }
}
```

## Discovery

Businesses can follow standard advertisement mechanism to advertise loyalty support in the Business profile. Currently the loyalty extension can decorate both the cart and checkout capabilities. Businesses MAY advertise loyalty support for cart only, checkout only, or both. Platforms SHOULD check which resources are extended.

```json
{
  "ucp": {
    "version": "2026-04-08",
    "capabilities": {
      "dev.ucp.shopping.cart": [
        {
          "version": "2026-04-08",
          "spec": "https://ucp.dev/2026-04-08/specification/cart",
          "schema": "https://ucp.dev/2026-04-08/schemas/shopping/cart.json"
        }
      ],
      "dev.ucp.shopping.checkout": [
        {
          "version": "2026-04-08",
          "spec": "https://ucp.dev/2026-04-08/specification/checkout",
          "schema": "https://ucp.dev/2026-04-08/schemas/shopping/checkout.json"
        }
      ],
      "dev.ucp.common.loyalty": [
        {
          "version": "2026-04-08",
          "extends": ["dev.ucp.shopping.cart", "dev.ucp.shopping.checkout"],
          "spec": "https://ucp.dev/2026-04-08/specification/loyalty",
          "schema": "https://ucp.dev/2026-04-08/schemas/common/loyalty.json"
        }
      ]
    }
  }
}
```

**Dependencies:**

* Cart Capability
* Checkout Capability

## Schema

### Entities

#### Loyalty

{{ extension_schema_fields('loyalty.json#/$defs/loyalty', 'loyalty') }}

#### Loyalty Membership

{{ extension_schema_fields('loyalty.json#/$defs/loyalty_membership', 'loyalty') }}

#### Membership Tier

{{ extension_schema_fields('loyalty.json#/$defs/membership_tier', 'loyalty') }}

#### Membership Tier Benefit

{{ extension_schema_fields('loyalty.json#/$defs/membership_tier_benefit', 'loyalty') }}

#### Membership Reward

{{ extension_schema_fields('loyalty.json#/$defs/membership_reward', 'loyalty') }}

#### Reward Currency

{{ extension_schema_fields('loyalty.json#/$defs/reward_currency', 'loyalty') }}

#### Earning Forecast

{{ extension_schema_fields('loyalty.json#/$defs/earning_forecast', 'loyalty') }}

#### Earning Breakdown

{{ extension_schema_fields('loyalty.json#/$defs/earning_breakdown', 'loyalty') }}

## Loyalty behavior

The loyalty extension holds a key-value map whose keys are reverse-domain identifiers — same convention as services, capabilities, and payment handlers in the business profile, and represent eligibility claims about loyalty memberships that businesses recognize. The values contain detailed membership info corresponding to the claims, and specifically contains a required `provisional` field to indicate the verification state.

Programs that can be joined independently MUST be modeled as separate sibling entries under the loyalty map, distinguished by their reverse-domain naming.

Platforms MAY send buyer loyalty membership claims via `context.eligibility` in the request to activate loyalty extension and claim for loyalty benefits. Alternatively, when the buyer is authenticated and the business can determine loyalty membership from the authenticated identity, businesses MAY populate the loyalty extension without an explicit eligibility claim. In this case, the map key MUST be the same reverse-domain identifier the business would accept as a claim value.

* When a business verifies a membership claim or determines membership from authenticated identity, it MUST return `provisional: false`. It also MUST populate the active tier(s) the buyer holds within the `tiers` array and SHOULD set `display_id` as a masked unique identifier of the buyer.
* When a membership claim in the request is recognized and accepted but not verified by the business, the business MUST return `provisional: true`. It MUST NOT populate the `tiers` array (i.e. leave it empty) and MUST NOT return `display_id`.
* When a membership claim in the request is accepted but cannot be verified, the business MUST communicate the failure via a recoverable `message` with `type: "error"` and `code: "eligibility_invalid"`. Platforms MAY then choose to remove the membership claim and proceed the checkout without loyalty benefits applied.

At checkout completion, all accepted but unverified loyalty claims MUST be resolved per the [Eligibility Verification at Completion](checkout.md#eligibility-verification-at-completion) contract defined in the checkout capability.

### Monetary loyalty benefits

When monetary price impacting loyalty benefits (e.g. member pricing/shipping) are available, it is worth noting that sometimes they have extra conditions besides membership requirement (e.g. Save $10 with $500+ purchase for members). In this case, businesses MUST check the eligibility of these extra conditions first regardless of membership claims/authorization identity. If the check passes or none is needed, businesses MUST surface the price-impacting loyalty benefits via the base capability's `totals` / `line_items[].totals`, and use `type: "items_discount"` with `display_text` to attribute the loyalty source when possible. If discount extension is also available, businesses MAY additionally populate `discounts.applied[]` with `eligibility` set to hold the corresponding membership claim(s) that are required for the discount (some loyalty discount is unlocked only when the buyer holds multiple memberships simultaneously). Because the discount’s eligibility relies on the conjunctive intersection of all eligibility keys, the `discounts.applied[]` object MUST evaluate to `provisional: true` if any string listed within `eligibility` field points to a loyalty membership where `provisional` is true. Conversely, disjunctive (any-of) eligibility — where any one of several independent claims unlocks the same benefit — MUST NOT be combined into a single array. Instead, businesses MUST model these alternative paths by emitting completely separate objects inside `discounts.applied[]`, ensuring each separate path carries its own discrete string mapping. This allows the platform to correctly and easily identify discount applicability and render it to buyers. If the check fails, businesses SHOULD notify the buyer via messages with `type: "warning"` and explain the inapplicability of those monetary loyalty benefits.

When loyalty membership claims are accepted, business MAY use `type: "info"` to explain the effects of applied monetary loyalty benefits .

**Loyalty benefits message codes:**

| Type      | Code                           | When                                                    |
| --------- | ------------------------------ | ------------------------------------------------------- |
| `info`    | `membership_benefit_eligible`  | Specific benefit confirmed applicable to this order     |
| `warning` | `membership_benefit_ineligible`| Benefit exists but conditions not met for this order    |

Building on the store loyalty card example from [Eligibility Verification at Completion](checkout.md#eligibility-verification-at-completion), and assume it offers one unconditional pricing discount on the product and one conditional discount that current checkout cart fails to satisfy. Platform can surface the first provisional discount with disclaimers like "verified at purchase" and additionally show a warning message to disclose the inapplicability of the second discount.

=== "Request"

    ```json
    {
      "context": {
        "eligibility": ["com.example.loyalty.visa_card"]
      },
      "line_items": [
        {
          "item": {
            "id": "prod_1",
            "quantity": 1,
            "title": "T-Shirt",
            "price": 1000
          }
        }
      ]
    }
    ```

=== "Response"

    ```json
    {
      "discounts": {
        "applied": [
          {
            "title": "Loyalty benefit 1",
            "amount": 10,
            "provisional": true,
            "eligibility": "com.example.loyalty.visa_card",
            "allocations": [
              {"path": "$.line_items[0]", "amount": 10}
            ]
          }
        ]
      },
      "loyalty": {
        "com.example.loyalty.visa_card": {
          "id": "membership_1",
          "name": "My Loyalty Program",
          "provisional": true
        }
      },
      "messages": [
        {
          "type": "warning",
          "code": "membership_benefit_ineligible",
          "path": "$.loyalty['com.example.loyalty.visa_card']",
          "content": "Cart size is smaller than required to receive the $10 discount."
        }
      ]
    }
    ```

Buyer can proceed with checkout without any cart update and if the claim is verified successfully by the business, the unconditional member pricing discount becomes non-provisional, and `display_id` is returned.

=== "Request"

    ```json
    {
      "context": {
        "eligibility": ["com.example.loyalty.visa_card"]
      },
      "line_items": [
        {
          "item": {
            "id": "prod_1",
            "quantity": 1,
            "title": "T-Shirt",
            "price": 1000
          }
        }
      ]
    }
    ```

=== "Response"

    ```json
    {
      "discounts": {
        "applied": [
          {
            "title": "Loyalty benefit 1",
            "amount": 10,
            "provisional": false,
            "eligibility": "com.example.loyalty.visa_card",
            "allocations": [
              {"path": "$.line_items[0]", "amount": 10}
            ]
          }
        ]
      },
      "loyalty": {
        "com.example.loyalty.visa_card": {
          "id": "membership_1",
          "display_id": "****5678",
          "name": "My Loyalty Program",
          "tiers": [
            {
              "id": "tier_1",
              "name": "Loyalty Visa Holder",
              "benefits": [
                { "id": "BEN_001", "description": "Early access to sales" }
              ]
            }
          ],
          "provisional": false
        }
      }
    }
    ```

If the claim can not be verified, a recoverable error should be returned from business via `messages[]`. Businesses MAY set the optional `path` field within to support back referencing to the membership metadata that corresponds to the claim. In this case, the loyalty extension needs to be included in the response as well.

=== "Request"

    ```json
    {
      "context": {
        "eligibility": ["com.example.loyalty.visa_card"]
      },
      "line_items": [
        {
          "item": {
            "id": "prod_1",
            "quantity": 1,
            "title": "T-Shirt",
            "price": 1000
          }
        }
      ]
    }
    ```

=== "Response"

    ```json
    {
      "messages": [
        {
          "type": "error",
          "severity": "recoverable",
          "code": "eligibility_invalid",
          "content": "Buyer is not a loyalty Visa holder."
        }
      ]
    }
    ```

## Use Cases and Examples

With the help of the loyalty extension, the cart/checkout capability can be further decorated to provide full visibility into buyers’ member-exclusive perks and allows the platform to render the extra information to facilitate the transaction.

### Compound Price-Impacting Benefits

Loyalty extension can provide buyer status info to allow the platform to transparently assert that correct and comprehensive member discounts are applied. In the example below, the buyer receives a 15% bonus discount because they hold BOTH the Retail Club membership and the Retail Card. The `eligibility` attribute reflects this conjunction natively. Platform now not only can explain the source of discount via `discounts.applied[].title` within discount extension, but also assure the buyer that member specific discount is recognized because of their verified loyalty status via `tiers[].name` within loyalty extension, which can be sourced from correlating `discounts.applied[].eligibility` to `loyalty['com.example.retail_club']` and `loyalty['com.example.retail_card']` to identify which memberships provide the monetary benefit. Platform can then render “Retail Club Gold Member and Retail Visa Card benefits applied.” for example.

=== "Request"

    ```json
    {
      "context": {
        "eligibility": [
          "com.example.retail_club",
          "com.example.retail_card"
        ]
      },
      "line_items": [
        {
          "item": {
            "id": "prod_1",
            "quantity": 1,
            "title": "T-Shirt",
            "price": 1000
          }
        }
      ]
    }
    ```

=== "Response"

    ```json
    {
      "line_items": [
        {
          "id": "li_1",
          "item": {
            "id": "prod_1",
            "quantity": 1,
            "title": "T-Shirt",
            "price": 1000
          },
          "totals": [
            {"type": "subtotal", "amount": 1000},
            {"type": "items_discount", "display_text": "Loyalty member benefit", "amount": -150},
            {"type": "total", "amount": 850}
          ]
        }
      ],
      "discounts": {
        "applied": [
          {
            "title": "Club Member + Cardholder 15% Bonus",
            "amount": 150,
            "method": "each",
            "provisional": false,
            "eligibility": [
              "com.example.retail_club",
              "com.example.retail_card"
            ],
            "allocations": [
              {"path": "$.line_items[0]", "amount": 150}
            ]
          }
        ]
      },
      "loyalty": {
        "com.example.retail_club": {
          "id": "membership_1",
          "display_id": "****5678",
          "name": "Retail Club",
          "provisional": false,
          "tiers": [
            {
              "id": "gold",
              "name": "Gold Member",
              "benefits": [
                { "id": "BEN_001", "description": "Early access to sales" }
              ]
            }
          ]
        },
        "com.example.retail_card": {
          "id": "membership_2",
          "display_id": "****1234",
          "name": "Retail Card",
          "provisional": false,
          "tiers": [
            {
              "id": "cardholder",
              "name": "Retail Visa Card",
              "benefits": [
                { "id": "BEN_002", "description": "Free standard shipping" }
              ]
            }
          ]
        }
      },
      "totals": [
        {"type": "subtotal", "display_text": "Subtotal", "amount": 1000},
        {"type": "items_discount", "display_text": "Loyalty member benefit", "amount": -150},
        {"type": "total", "display_text": "Estimated Total", "amount": 850}
      ]
    }
    ```

### Reward Earnings Forecast

In addition to immediate-value benefits like member pricing/shipping, delayed-value collectable reward benefits are another crucial element within the loyalty ecosystem. Displaying earnings forecasts of these rewards before the buyer commits complements and to some extent helps agents handle price objections - rewards earning becomes additional value on top of any pricing discount. In this example, businesses provide the reward earning forecast with a breakdown, using `benefit_id` to correlate the specific `membership_tier_benefit` that produced the rule and giving platforms a way to explain with full transparency on why the buyer is earning and how the earning is calculated.

=== "Request"

    ```json
    {
      "context": {
        "eligibility": ["com.example.retail_club"]
      },
      "line_items": [
        {
          "item": {
            "id": "prod_1",
            "quantity": 1,
            "title": "T-Shirt",
            "price": 1000
          }
        }
      ]
    }
    ```

=== "Response"

    ```json
    {
      "line_items": [
        {
          "id": "li_1",
          "item": {
            "id": "prod_1",
            "quantity": 1,
            "title": "T-Shirt",
            "price": 1000
          },
          "totals": [
            {"type": "subtotal", "amount": 1000},
            {"type": "total", "amount": 1000}
          ]
        }
      ],
      "loyalty": {
        "com.example.retail_club": {
          "id": "membership_1",
          "display_id": "****5678",
          "name": "Retail Club",
          "provisional": false,
          "tiers": [
            {
              "id": "gold",
              "name": "Gold",
              "benefits": [
                { "id": "BEN_001", "description": "1 point per $1 on everything" },
                { "id": "BEN_002", "description": "2 extra point/dollar on footwear" }
              ]
            }
          ],
          "rewards": [
            {
              "currency": {
                "name": "LoyaltyStars",
                "code": "LST"
              },
              "earning_forecast": {
                "amount": 30,
                "breakdown": [
                  {
                    "id": "RULE_1",
                    "description": "1 point/dollar on everything",
                    "amount": 10,
                    "benefit_id": "BEN_001"
                  },
                  {
                    "id": "RULE_2",
                    "description": "2 extra point/dollar on footwear",
                    "amount": 20,
                    "benefit_id": "BEN_002"
                  }
                ]
              }
            }
          ]
        }
      },
      "totals": [
        {"type": "subtotal", "display_text": "Subtotal", "amount": 1000},
        {"type": "total", "display_text": "Estimated Total", "amount": 1000}
      ]
    }
    ```

## Implementation guidelines

* Loyalty extension response MUST be data-minimized and MUST NOT expose raw stable member identifiers (as this would allow the platform to uniquely identifier individual buyer)
* Loyalty extension response MUST only include `display_id` after verified/authenticated membership
* Loyalty extension response MUST treat all `context.eligibility` values in the request as buyer claims rather than proof
