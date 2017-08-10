# Contributing

We welcome all contributions - send a pull request or open an issue. When possible, send different pull requests by section/topic.

Feel free to peruse the [open issues](../issues) for ideas which need to be expanded on a bit here.

### Layout and Compilation

We aren't using the `old-src` folder anymore. You should directly edit the README.md instead.

### Audience

Write for an intermediate Ethereum developer, they know the basics of Solidity programming and have coded a number of contracts

### Style Guidelines

#### General

- **Favor succinctness in writing**
  - Use max 3-4 sentences in a section (exceptions can be made when critical)
  - Show, don’t tell (examples speak more than lengthy exposition)
  - Include a simple, illustrative example rather than complex examples that require substantial, extraneous reading
- **Add a source link to the original document when available**
- **Create new sections when warranted**
- **Keep code lines under 80 characters when possible**
- **Mark code** as insecure, bad, good where relevant
- Use the format of the [Airbnb Javascript Style guide](https://github.com/airbnb/javascript) as a starting point

#### Recommendations Section

- Always favor a declarative tip starting with a verb for the section title
- Include good and bad examples, when possible
- Ensure each subsection has an anchor tag for future hyperlinking

#### Attacks Section

- Provide an example - then point to a recommendation for the solution in the relevant section of the doc
- List first/most visible attack, where possible
- Ensure each subsection has an anchor tag for future hyperlinking
- Mark vulnerable pieces of code as `// INSECURE`

