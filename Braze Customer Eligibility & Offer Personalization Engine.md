{% comment %}
============================================================
BRAZE CUSTOMER ELIGIBILITY & OFFER PERSONALIZATION ENGINE
============================================================

Purpose:
- Retrieve customer profile
- Integrate CRM and transaction data
- Retrieve loyalty information
- Evaluate customer eligibility
- Apply business rules
- Select the most relevant offer
- Provide fallback personalization
============================================================
{% endcomment %}


{% comment %}
------------------------------------------------------------
1. CORE BRAZE PROFILE
------------------------------------------------------------
{% endcomment %}

{% assign user_id = ${user_id} %}
{% assign first_name = ${first_name} | default: "Customer" %}
{% assign email = ${email_address} | default: "" %}
{% assign country = ${country} | default: "NG" %}

{% assign first_name = first_name | strip %}
{% assign email = email | strip | downcase %}


{% comment %}
------------------------------------------------------------
2. USER PROFILE ATTRIBUTES
------------------------------------------------------------
{% endcomment %}

{% assign customer_type =
  ${custom_attribute.${customer_type}}
  | default: "individual"
%}

{% assign monthly_spend =
  ${custom_attribute.${monthly_spend}}
  | default: 0
%}

{% assign preferred_category =
  ${custom_attribute.${preferred_category}}
  | default: ""
%}


{% comment %}
------------------------------------------------------------
3. CRM SYSTEM
------------------------------------------------------------
Retrieve account status and customer information.
------------------------------------------------------------
{% endcomment %}

{% connected_content
  https://crm.example.com/api/v2/customers/{{user_id}}
  :method get
  :headers {
    "Authorization": "Bearer CRM_API_KEY",
    "Content-Type": "application/json"
  }
  :cache_max_age 300
  :save customer
%}


{% assign account_status =
  customer.account.status
  | default: "inactive"
%}

{% assign customer_since =
  customer.account.created_at
%}

{% assign marketing_consent =
  customer.preferences.marketing_consent
  | default: false
%}


{% comment %}
------------------------------------------------------------
4. TRANSACTION SYSTEM
------------------------------------------------------------
Retrieve recent purchasing behavior.
------------------------------------------------------------
{% endcomment %}

{% connected_content
  https://payments.example.com/api/customers/{{user_id}}/summary
  :method get
  :headers {
    "Authorization": "Bearer PAYMENT_API_KEY",
    "Content-Type": "application/json"
  }
  :cache_max_age 300
  :save transactions
%}


{% assign transaction_count =
  transactions.total_transactions
  | default: 0
%}

{% assign total_spend =
  transactions.total_spend
  | default: 0
%}

{% assign last_transaction =
  transactions.last_transaction
%}


{% comment %}
------------------------------------------------------------
5. LOYALTY SYSTEM
------------------------------------------------------------
------------------------------------------------------------
{% endcomment %}

{% connected_content
  https://loyalty.example.com/api/v2/members/{{user_id}}
  :method get
  :headers {
    "Authorization": "Bearer LOYALTY_API_KEY",
    "Content-Type": "application/json"
  }
  :cache_max_age 300
  :save member
%}


{% assign loyalty_points =
  member.points
  | default: 0
%}

{% assign loyalty_tier =
  member.tier
  | default: "standard"
%}


{% comment %}
------------------------------------------------------------
6. DETERMINE CUSTOMER VALUE
------------------------------------------------------------
------------------------------------------------------------
{% endcomment %}

{% assign customer_score = 0 %}


{% if total_spend >= 1000000 %}
  {% assign customer_score = customer_score | plus: 50 %}

{% elsif total_spend >= 500000 %}
  {% assign customer_score = customer_score | plus: 35 %}

{% elsif total_spend >= 100000 %}
  {% assign customer_score = customer_score | plus: 20 %}
{% endif %}


{% if transaction_count >= 20 %}
  {% assign customer_score = customer_score | plus: 30 %}

{% elsif transaction_count >= 10 %}
  {% assign customer_score = customer_score | plus: 20 %}

{% elsif transaction_count >= 5 %}
  {% assign customer_score = customer_score | plus: 10 %}
{% endif %}


{% if loyalty_tier == "platinum" %}
  {% assign customer_score = customer_score | plus: 30 %}

{% elsif loyalty_tier == "gold" %}
  {% assign customer_score = customer_score | plus: 20 %}

{% elsif loyalty_tier == "silver" %}
  {% assign customer_score = customer_score | plus: 10 %}
{% endif %}


{% comment %}
------------------------------------------------------------
7. CUSTOMER SEGMENT
------------------------------------------------------------
------------------------------------------------------------
{% endcomment %}

{% assign segment = "standard" %}

{% if customer_score >= 80 %}
  {% assign segment = "premium" %}

{% elsif customer_score >= 50 %}
  {% assign segment = "high_value" %}

{% elsif customer_score >= 25 %}
  {% assign segment = "active" %}

{% elsif transaction_count == 0 %}
  {% assign segment = "new_customer" %}
{% endif %}


{% comment %}
------------------------------------------------------------
8. OFFER ELIGIBILITY
------------------------------------------------------------
------------------------------------------------------------
{% endcomment %}

{% assign eligible = true %}
{% assign exclusion_reason = "" %}


{% if account_status != "active" %}

  {% assign eligible = false %}
  {% assign exclusion_reason = "inactive_account" %}

{% elsif marketing_consent != true %}

  {% assign eligible = false %}
  {% assign exclusion_reason = "marketing_not_permitted" %}

{% endif %}


{% comment %}
------------------------------------------------------------
9. OFFER SELECTION ENGINE
------------------------------------------------------------
------------------------------------------------------------
{% endcomment %}

{% assign offer = "WELCOME_OFFER" %}
{% assign discount = 10 %}


{% if segment == "premium" %}

  {% assign offer = "VIP_EXCLUSIVE" %}
  {% assign discount = 30 %}


{% elsif segment == "high_value" %}

  {% assign offer = "LOYALTY_REWARD" %}
  {% assign discount = 20 %}


{% elsif segment == "active" %}

  {% assign offer = "CUSTOMER_REWARD" %}
  {% assign discount = 15 %}


{% elsif segment == "new_customer" %}

  {% assign offer = "WELCOME_OFFER" %}
  {% assign discount = 10 %}

{% endif %}


{% comment %}
------------------------------------------------------------
10. CATEGORY PERSONALIZATION
------------------------------------------------------------
------------------------------------------------------------
{% endcomment %}

{% assign category_message = "" %}

{% if preferred_category != blank %}

  {% assign category_message =
    "We selected this offer based on your interest in "
    | append: preferred_category
    | append: "."
  %}

{% endif %}


{% comment %}
------------------------------------------------------------
11. FINAL MESSAGE
------------------------------------------------------------
------------------------------------------------------------
{% endcomment %}

{% if eligible == false %}

  {% abort_message(
    "Customer excluded: "
    | append: exclusion_reason
  ) %}


{% elsif offer == "VIP_EXCLUSIVE" %}

  Hi {{first_name}},

  As one of our most valued customers, you've unlocked
  an exclusive {{discount}}% offer.

  {{category_message}}

  Your current loyalty tier is {{loyalty_tier}}.


{% elsif offer == "LOYALTY_REWARD" %}

  Hi {{first_name}},

  Thank you for being a loyal customer.

  You've unlocked {{discount}}% off your next purchase.

  You currently have {{loyalty_points}} loyalty points.


{% elsif offer == "CUSTOMER_REWARD" %}

  Hi {{first_name}},

  We appreciate having you as a customer.

  Here's {{discount}}% off your next purchase.

  {{category_message}}


{% else %}

  Hi {{first_name}},

  Welcome! We've created a special {{discount}}% offer
  just for you.

{% endif %}