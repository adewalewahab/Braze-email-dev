{% assign user_id = ${user_id} %}
{% assign first_name = ${first_name} | default: "Customer" %}
{% assign country = ${country} | default: "NG" %}

{% assign spend = ${custom_attribute.${lifetime_value}} | default: 0 %}
{% assign orders = ${custom_attribute.${order_count}} | default: 0 %}
{% assign engagement = ${custom_attribute.${engagement_score}} | default: 0 %}
{% assign loyalty = ${custom_attribute.${loyalty_tier}} | default: "standard" %}

{% connected_content
  https://offers.example.com/api/users/{{user_id}}
  :method get
  :headers {
    "Authorization": "Bearer API_KEY"
  }
  :cache_max_age 300
  :save offer_data
%}

{% assign recommended_offer = offer_data.offer %}
{% assign eligible = offer_data.eligible | default: false %}

{% assign offer_type = "STANDARD" %}
{% assign discount = 10 %}

{% if spend >= 1000000 and loyalty == "platinum" %}
  {% assign offer_type = "VIP" %}
  {% assign discount = 30 %}

{% elsif spend >= 500000 %}
  {% assign offer_type = "PREMIUM" %}
  {% assign discount = 25 %}

{% elsif orders >= 10 %}
  {% assign offer_type = "LOYALTY" %}
  {% assign discount = 20 %}

{% elsif engagement >= 80 %}
  {% assign offer_type = "ENGAGEMENT" %}
  {% assign discount = 15 %}
{% endif %}

{% if eligible == false %}
  {% abort_message("Customer is not eligible for offer") %}
{% endif %}

Hi {{first_name}},

{% if offer_type == "VIP" %}

As one of our VIP customers, you've unlocked an
exclusive {{discount}}% offer.

{% elsif offer_type == "PREMIUM" %}

Because you're one of our valued customers, we've
selected a special {{discount}}% offer for you.

{% elsif offer_type == "LOYALTY" %}

Thank you for being a loyal customer.

Enjoy {{discount}}% off your next purchase.

{% elsif offer_type == "ENGAGEMENT" %}

We noticed you've been enjoying our products.

Here's {{discount}}% off your next purchase.

{% else %}

Here's a special {{discount}}% offer for you.

{% endif %}