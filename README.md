business-rules
==============

[![Build Status](https://travis-ci.com/amitkpandey-in/business-rules.svg?token=VFT53t8Uf9YtKaqpfH4N&branch=master)](https://travis-ci.com/amitkpandey-in/business-rules)

This is a fork of [yoyowallet/business-rules](https://github.com/yoyowallet/business-rules) which is fork of [venmo/business-rules](https://github.com/venmo/business-rules) from version 1.0.1. [yoyowallet/business-rules](https://github.com/yoyowallet/business-rules) is being maintained, hence forked this.

As a software system grows in complexity and usage, it can become burdensome if
every change to the logic/behavior of the system also requires you to write and
deploy new code. The goal of this business rules engine is to provide a simple
interface allowing anyone to capture new rules and logic defining the behavior
of a system, and a way to then process those rules on the backend.

You might, for example, find this is a useful way for analysts to define
marketing logic around when certain customers or items are eligible for a
discount or to automate emails after users enter a certain state or go through
a particular sequence of events.

<p align="center">
    <img src="http://cdn.memegenerator.net/instances/400x/36514579.jpg" />
</p>

## Usage

### 1. Define Your set of variables

Variables represent values in your system, usually the value of some particular object.  You create rules by setting threshold conditions such that when a variable is computed that triggers the condition some action is taken.

You define all the available variables for a certain kind of object in your code, and then later dynamically set the conditions and thresholds for those.

For example:

```python
class ProductVariables(BaseVariables):

    def __init__(self, product):
        self.product = product

    @numeric_rule_variable
    def current_inventory(self):
        return self.product.current_inventory

    @numeric_rule_variable(label='Days until expiration')
    def expiration_days(self)
        last_order = self.product.orders[-1]
        return (last_order.expiration_date - datetime.date.today()).days

    @string_rule_variable()
    def current_month(self):
        return datetime.datetime.now().strftime("%B")

    @select_rule_variable(options=Products.top_holiday_items())
    def goes_well_with(self):
        return products.related_products

    @numeric_rule_variable(params=[{
        'field_type': FIELD_NUMERIC,
        'name': 'days',
        'label': 'Days'
    }])
    def orders_sold_in_last_x_days(self, days):
        count = 0
        for order in self.product.orders:
            if (datetime.date.today() - order.date_sold).days < days:
                count += 1
        return count
```

### 2. Define your set of actions

These are the actions that are available to be taken when a condition is triggered.

For example:

```python
class ProductActions(BaseActions):

    def __init__(self, product):
        self.product = product

    @rule_action(params={"sale_percentage": FIELD_NUMERIC})
    def put_on_sale(self, sale_percentage):
        self.product.price = (1.0 - sale_percentage) * self.product.price
        self.product.save()

    @rule_action(params={"number_to_order": FIELD_NUMERIC})
    def order_more(self, number_to_order):
        ProductOrder.objects.create(product_id=self.product.id,
                                    quantity=number_to_order)
```

If you need a select field for an action parameter, another -more verbose- syntax is available:

```python
class ProductActions(BaseActions):

    def __init__(self, product):
        self.product = product

    @rule_action(params=[{'fieldType': FIELD_SELECT,
                          'name': 'stock_state',
                          'label': 'Stock state',
                          'options': [
                            {'label': 'Available', 'name': 'available'},
                            {'label': 'Last items', 'name': 'last_items'},
                            {'label': 'Out of stock', 'name': 'out_of_stock'}
                        ]}])
    def change_stock_state(self, stock_state):
        self.product.stock_state = stock_state
        self.product.save()
```

If you introduced a new action parameter but already have an older set of active Rules,
which will all require to be updated with this new parameter, you can use `ActionParam` class which
contains default value for parameter along with field type:
```python
from business_rules.actions import ActionParam, BaseActions, rule_action
from business_rules import fields

class ProductActions(BaseActions):

    def __init__(self, product):
        self.product = product

    @rule_action(params={"sale_percentage": fields.FIELD_NUMERIC,
                         "on_sale": ActionParam(field_type=fields.FIELD_BOOLEAN, default_value=False})
    def put_on_sale(self, sale_percentage, on_sale):
        self.product.price = (1.0 - sale_percentage) * self.product.price
        self.on_sale = on_sale
        self.product.save()

```
### 3. Build the rules

A rule is just a JSON object that gets interpreted by the business-rules engine.

Note that the JSON is expected to be auto-generated by a UI, which makes it simple for anyone to set and tweak business rules without knowing anything about the code.  The javascript library used for generating these on the web can be found [here](https://github.com/venmo/business-rules-ui).

An example of the resulting python lists/dicts is:

```python
rules = [
# expiration_days < 5 AND current_inventory > 20
{ "conditions": { "all": [
      { "name": "expiration_days",
        "operator": "less_than",
        "value": 5,
      },
      { "name": "current_inventory",
        "operator": "greater_than",
        "value": 20,
      },
  ]},
  "actions": [
      { "name": "put_on_sale",
        "params": {"sale_percentage": 0.25},
      },
  ],
},

# current_inventory < 5 OR (current_month = "December" AND current_inventory < 20)
{ "conditions": { "any": [
            {
                "name": "current_inventory",
                "operator": "less_than",
                "value": 5,
            },
            {
                "all": [
                    {
                        "name": "current_month",
                        "operator": "equal_to",
                        "value": "December",
                    },
                    {
                        "name": "current_inventory",
                        "operator": "less_than",
                        "value": 20,
                    }
                ]
            }
        ]
  },
  "actions": [
    { "name": "order_more",
      "params":{"number_to_order": 40},
    },
  ],
},
# orders_sold_in_last_x_days(5) > 10
{
    "conditions": { "all": [
        {
            "name": "orders_sold_in_last_x_days",
            "operator": "greater_than",
            "value": 10,
            "params": {"days": 5},
        }
    ]},
    "actions": [
        {
            "name": "order_more",
            "fields": [{"name": "number_to_order", "value": 40}]
        }
    ]
}]
```

### Export the available variables, operators and actions

To e.g. send to your client so it knows how to build rules

```python
from business_rules import export_rule_data
export_rule_data(ProductVariables, ProductActions)
```

that returns

```json
{
  "variables": [
    {
      "name": "expiration_days",
      "label": "Days until expiration",
      "field_type": "numeric",
      "options": [],
      "params": []
    },
    {
      "name": "current_month",
      "label": "Current Month",
      "field_type": "string",
      "options": [],
      "params": []
    },
    {
      "name": "goes_well_with",
      "label": "Goes Well With",
      "field_type": "select",
      "options": [
        "Eggnog",
        "Cookies",
        "Beef Jerkey"
      ],
      "params": []
    },
    {
      "name": "orders_sold_in_last_x_days",
      "label": "Orders Sold In Last X Days",
      "field_type": "numeric",
      "options": [],
      "params": [
        {
          "field_type": "numeric",
          "name": "days",
          "label": "Days"
        }
      ]
    }
  ],
  "actions": [
    {
      "name": "put_on_sale",
      "label": "Put On Sale",
      "params": {
        "sale_percentage": "numeric"
      }
    },
    {
      "name": "order_more",
      "label": "Order More",
      "params": {
        "number_to_order": "numeric"
      }
    }
  ],
  "variable_type_operators": {
    "numeric": [
      {
        "name": "equal_to",
        "label": "Equal To",
        "input_type": "numeric"
      },
      {
        "name": "less_than",
        "label": "Less Than",
        "input_type": "numeric"
      },
      {
        "name": "greater_than",
        "label": "Greater Than",
        "input_type": "numeric"
      }
    ],
    "string": [
      {
        "name": "equal_to",
        "label": "Equal To",
        "input_type": "text"
      },
      {
        "name": "non_empty",
        "label": "Non Empty",
        "input_type": "none"
      }
    ]
  }
}
```

To validate rule data:

```python
from business_rules import validate_rule_data
is_valid = validate_rule_data(ProductVariables, ProductActions, {'conditions':[], 'actions':[]})
```

### Run your rules

```python
from business_rules import run_all

rules = _some_function_to_receive_from_client()

for product in Products.objects.all():
    run_all(rule_list=rules,
            defined_variables=ProductVariables(product),
            defined_actions=ProductActions(product),
            stop_on_first_trigger=True,
           )
```

## API

### Variable Types and Decorators:

The type represents the type of the value that will be returned for the variable and is necessary since there are different available comparison operators for different types, and the front-end that's generating the rules needs to know which operators are available.

All decorators can optionally take a label:
- `label` - A human-readable label to show on the frontend. By default we just split the variable name on underscores and capitalize the words.
- `params` - A list of parameters that will be passed to the variable when its value is calculated. The list elements
should be dictionaries with a `field_type` to specify the type and `name` that corresponds to an argument of the
variable function.

The available types and decorators are:

#### `numeric` - an integer, float, or python Decimal.

`@numeric_rule_variable` operators:

* `equal_to`
* `greater_than`
* `less_than`
* `greater_than_or_equal_to`
* `less_than_or_equal_to`

Note: to compare floating point equality we just check that the difference is less than some small epsilon

#### `string` - a python bytestring or unicode string.

`@string_rule_variable` operators:

* `equal_to`
* `starts_with`
* `ends_with`
* `contains`
* `matches_regex`
* `non_empty`

#### `boolean` - a True or False value.

`@boolean_rule_variable` operators:

* `is_true`
* `is_false`

#### `select` - a set of values, where the threshold will be a single item.

`@select_rule_variable` operators:

* `contains`
* `does_not_contain`

#### `select_multiple` - a set of values, where the threshold will be a set of items.

`@select_multiple_rule_variable` operators:

* `contains_all`
* `is_contained_by`
* `shares_at_least_one_element_with`
* `shares_exactly_one_element_with`
* `shares_no_elements_with`

#### `datetime` - a Timestamp value

A rule variable accepts the following types of values:

* int
* string with format `%Y-%m-%dT%H:%M:%S`
* string with format `%Y-%m-%d`

A variable can return the following types of values:

* int
* datetime
* date
* string with format `%Y-%m-%dT%H:%M:%S`
* string with format `%Y-%m-%d`

`@datetime_rule_variable` operators:

* `equal_to`
* `before_than`
* `before_than_or_equal_to`
* `after_than`
* `after_than_or_equal_to`


#### `time` - a Time value

A rule variable accepts the following types of values:

* string with format `%H:%M:%S`
* string with format `%H:%M`

A variable can return the following types of values:

* datetime
* time
* string with format `%H:%M:%S`
* string with format `%H:%M`

`@time_rule_variable` operators:

* `equal_to`
* `before_than`
* `before_than_or_equal_to`
* `after_than`
* `after_than_or_equal_to`


## Contributing

Open up a pull request, making sure to add tests for any new functionality. To set up the dev environment (assuming you're using [virtualenvwrapper](http://docs.python-guide.org/en/latest/dev/virtualenvs/#virtualenvwrapper)):

```bash
$ mkvirtualenv business-rules
$ pip install -r dev-requirements.txt
$ nosetests
```
