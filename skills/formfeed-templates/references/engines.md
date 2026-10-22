# The three template languages

Every template is written in one language, the `engine` of its `template.json`: `jinja2`, `liquid`
or `handlebars`. They share the data, the helpers and the settings; only the syntax differs. Never
mix two in one template.

|                       | Jinja2                                | Liquid                                    | Handlebars                                |
| --------------------- | ------------------------------------- | ----------------------------------------- | ----------------------------------------- |
| Variable              | `{{ invoice.number }}`                | `{{ invoice.number }}`                    | `{{invoice.number}}`                      |
| Helper with arguments | `{{ total \| money("EUR") }}`         | `{{ total \| money: "EUR" }}`             | `{{money total "EUR"}}`                   |
| Helper with options   | `{{ qrcode(url, { size: 120 }) }}`    | `{{ url \| qrcode: size: 120 }}`          | `{{qrcode url size=120}}`                 |
| Loop                  | `{% for l in lines %}…{% endfor %}`   | `{% for l in lines %}…{% endfor %}`       | `{{#each lines}}…{{/each}}`               |
| Index from 1          | `{{ loop.index }}`                    | `{{ forloop.index }}`                     | `{{add @index 1}}`                        |
| Condition             | `{% if a %}…{% elif b %}…{% endif %}` | `{% if a %}…{% elsif b %}…{% endif %}`    | `{{#if a}}…{{else if b}}…{{/if}}`         |
| Comparison            | `{% if total > 1000 %}`               | `{% if total > 1000 %}`                   | `{{#if (gt total 1000)}}`                 |
| Assignment            | `{% set vat = total * 0.19 %}`        | `{% assign vat = total \| times: 0.19 %}` | none: compute in the data or with helpers |
| Include a partial     | `{% include "footer" %}`              | `{% include "footer" %}`                  | `{{> footer}}`                            |
| Comment               | `{# … #}`                             | `{% comment %}…{% endcomment %}`          | `{{!-- … --}}`                            |
| Escaping              | automatic                             | **none**: add `\| escape`                 | automatic                                 |
| HTML as it is         | `{{ html \| safe }}`                  | automatic                                 | `{{{html}}}`                              |

## The same invoice three times

```json data
{
  "invoice": { "number": "2026-0042", "date": "2026-09-11", "paid": false },
  "lines": [
    { "description": "Consulting", "qty": 8, "price": 120 },
    { "description": "Travel", "qty": 1, "price": 180 }
  ]
}
```

```jinja2
<h1>Invoice {{ invoice.number }}</h1>
<p>{{ invoice.date | date("dd.MM.yyyy") }}</p>
<table>
  {% for line in lines %}
  <tr>
    <td>{{ loop.index }}. {{ line.description }}</td>
    <td>{{ (line.qty * line.price) | money("EUR") }}</td>
  </tr>
  {% endfor %}
</table>
{% if not invoice.paid %}<p>Payable within 14 days.</p>{% endif %}
```

```liquid
<h1>Invoice {{ invoice.number | escape }}</h1>
<p>{{ invoice.date | date: "dd.MM.yyyy" }}</p>
<table>
  {% for line in lines %}
  <tr>
    <td>{{ forloop.index }}. {{ line.description | escape }}</td>
    <td>{{ line.qty | times: line.price | money: "EUR" }}</td>
  </tr>
  {% endfor %}
</table>
{% unless invoice.paid %}<p>Payable within 14 days.</p>{% endunless %}
```

```handlebars
<h1>Invoice {{invoice.number}}</h1>
<p>{{date invoice.date 'dd.MM.yyyy'}}</p>
<table>
  {{#each lines}}
    <tr>
      <td>{{add @index 1}}. {{description}}</td>
      <td>{{money (multiply qty price) 'EUR'}}</td>
    </tr>
  {{/each}}
</table>
{{#unless invoice.paid}}<p>Payable within 14 days.</p>{{/unless}}
```

## Calling a helper

A helper is the same function in every language. In Jinja2 it works as a function and as a filter,
and options are an object literal. In Liquid a helper is always a filter: one that needs no input
still takes one (`{{ '' | pageBreak }}`), and options are `name: value` pairs after the positional
arguments. In Handlebars arguments are positional and options are `name=value` pairs; helpers nest
in parentheses. The call of every helper in all three languages is in [helpers.md](helpers.md).

## What differs from the language you know

**Jinja2** runs on Nunjucks, the same in the preview and on the servers.

- `"%.2f" % value` prints `NaN`. Use `number(2)`, `money`, or `"{:,.2f}".format(value)`.
- Tests take parentheses: `is divisibleby(3)`. `loop.cycle()` does not exist: use `cycler()`.
- A `{% set %}` inside a loop changes a variable set before the loop, and the new value stays after
  it. `namespace()` with `{% set ns.total = … %}` works too.

**Liquid**

- It does not escape. Add `| escape` to every value from outside: names, addresses, free text.
- Dates take date-fns tokens (`dd.MM.yyyy`), not `strftime`. `%d.%m.%Y` prints garbage.
- Arithmetic is filters: `times`, `plus`, `minus`, `divided_by`. `7 | divided_by: 2` is `3.5`; add
  `| floor` for the integer.
- An unknown filter is an error, not an empty string.
- A filter cannot stand inside a condition. Assign first:
  `{% assign big = total | gt: 1000 %}{% if big %}…{% endif %}`.
- `{% render "footer" %}` sees only what is passed to it; `{% include "footer" %}` sees all data.

**Handlebars**

- No operators. Use `add`, `subtract`, `multiply`, `divide` and `eq`, `ne`, `gt`, `gte`, `lt`,
  `lte`, `and`, `or`, `not` in parentheses: `{{#if (gt total 1000)}}`.
- Inside `{{#each}}` names refer to the item. Reach the outer data with `../` or `@root`.
- `{{date}}` alone prints a field named `date` of the current item; with arguments it is the helper.

## Missing values

A variable or field that is not in the data prints as empty text in all three languages. An
unknown helper fails the template, and so does an include of a partial that does not exist.
`formfeed validate` warns about every variable the data set lacks, so check against data sets that
cover the optional fields.

Every language also sees the organisation's brand kit as `brand` (`{{ brand.legal_footer }}`),
unless the data has a field `brand`.
