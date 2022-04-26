# Slash-Premium-System-Djs
Premium System Djs v13
<p align="center">
  <a href="https://discord.gg/PcUVWApWN3" target="blank"><img src="https://cdn.discordapp.com/attachments/881491251784978452/925487363017277510/djs.png" width="200" height="200" alt="Djs" /></a>
</p>

# Premium.djs

This guide will explain you how to create a Premium System for your Discord.js Bot.

## Requirements

- Basic Knowledge of JavaScript/NodeJS
- Basic+ Knowledge of MongoDB/Mongoose
- Basic+ experience with NPM
- Good experience with Discord.js
- A working connection to MongoDB with Schemas

Discord.js v13 or higher.
Nodejs v16 or higher.

Understanding Pathing

```diff
/ = Root directory.
. = This location.
.. = Up a directory.
./ = Current directory.
../ = Parent of current directory.
../../ = Two directories backwards.
```

## Get started

Lets get started by installing some dependencies, open your favourite terminal.
Run the following Commands in your Terminal.

```shell
npm install voucher-code-generator
npm install moment
npm install discord.js@latest
npm install mongoose
npm install node-cron
```

Close your terminal, we won't need it while coding.
The next steps are very easy. Create a folder called
`Models`, the path would look like this: `src/Models`.
In there, create a file called `code.js`.

We now want Mongoose to store the Data we generate.

```js
const mongoose = require("mongoose");

// Generate Premium Code
const premiumCode = mongoose.Schema({
  code: {
    type: mongoose.SchemaTypes.String,
    default: null,
  },

  // Set the expire date and time. <Day, Week, Month, Year>
  expiresAt: {
    type: mongoose.SchemaTypes.Number,
    default: null,
  },

  // Set the plan <Day, Week, Month>.
  plan: {
    type: mongoose.SchemaTypes.String,
    default: null,
  },
});

module.exports = mongoose.model("premium-codes", premiumCode);
```

Great. Now we want to do the same for our Users/Members.
Create a file called `User.js` in the same Folder.
The path would look like this: `src/Models/User.js`

```js
const mongoose = require("mongoose");

// The heart of the User, here is everything saved that the User does.
// Such as Levels, Courses, Premium, Enrolled, XP, Leaderboard.
const user = mongoose.Schema({
  Id: {
    type: mongoose.SchemaTypes.String,
    required: true,
    unique: true,
  },
  isPremium: {
    type: mongoose.SchemaTypes.Boolean,
    default: false,
  },
  premium: {
    redeemedBy: {
      type: mongoose.SchemaTypes.Array,
      default: null,
    },

    redeemedAt: {
      type: mongoose.SchemaTypes.Number,
      default: null,
    },

    expiresAt: {
      type: mongoose.SchemaTypes.Number,
      default: null,
    },

    plan: {
      type: mongoose.SchemaTypes.String,
      default: null,
    },
  },
});
module.exports = mongoose.model("user", user);
```

Cool.
The next step is creating a code generator and a command to redeem them.
Let's start with generating a premium code for our users.
Go ahead and create a file called `generate.js` within your commands folder.
The path would look something like this: `src/commands/premium/generate.js`

```js
const { Command } = require("reconlx");
const ee = require("../../settings/embed.json");
const config = require("../../settings/config.json");
const moment = require("moment");
const voucher_codes = require("voucher-code-generator");
const schema = require("../../Models/code");

module.exports = new Command({
  // options
  name: "gencode",
  description: `gen premium codes`,
  userPermissions: [],
  category: "Premium",
  options: [
    {
      name: "plan",
      description: `choose your plan`,
      type: "STRING",
      required: true,
      choices: [
        {
          name: "daily",
          value: "daily",
        },
        {
          name: "weekly",
          value: "weekly",
        },
        {
          name: "monthly",
          value: "monthly",
        },
        {
          name: "yearly",
          value: "yearly",
        },
      ],
    },
    {
      name: "amount",
      description: `amount of codes`,
      type: "NUMBER",
      required: false,
    },
  ],
  // command start
  run: async ({ client, interaction, args }) => {
    // Code
    // As defined in the Schema, leave codes as an empty array variable
    let codes = [];

    // Display available plans of the code
    const plan = interaction.options.getString("plan");

    // Calculate time for the code to expire.
    let time;
    if (plan === "daily") time = Date.now() + 86400000;
    if (plan === "weekly") time = Date.now() + 86400000 * 7;
    if (plan === "monthly") time = Date.now() + 86400000 * 30;
    if (plan === "yearly") time = Date.now() + 86400000 * 365;

    // If the input is for ex. 10, generate 10 Codes. Default => 1 Code / Command.
    let amount = interaction.options.getNumber("amount");
    if (!amount) amount = 1;

    for (var i = 0; i < amount; i++) {
      const codePremium = voucher_codes.generate({
        pattern: "####-####-####",
      });

      // Save the Code as a String ("ABCDEF ...") in the Database
      const code = codePremium.toString().toUpperCase();

      // Security check, check if the code exists in the database.
      const find = await schema.findOne({
        code: code,
      });

      // If it does not exist, create it in the database.
      if (!find) {
        schema.create({
code: code,
          plan: plan,
          expiresAt: time,
        });

        // Push the new generated Code into the Queue
        codes.push(`${i + 1}- ${code}`);
      }
    }

    // Lastly, we want to send the new Code(s) into the Channel.
    interaction.followUp({
      content: `\`\`\`Generated +${codes.length}\n\n--------\n${codes.join(
        "\n"
      )}\n--------\n\nType - ${plan}\nExpires - ${moment(time).format(
        "dddd, MMMM Do YYYY"
      )}\`\`\`\nTo redeem, use \`/redeem <code>\``,
    });
  },
});
```

Perfect. Now our Bot is generating the Codes.
We now want to redeem it and save it under our Profile Settings in the Database.

Go ahead and create a file within your `commands` folder called `redeem.js`
The path would look like this: `src/commands/subfolder/redeem.js`

```js
const { Command } = require("reconlx");
const ee = require("../../settings/embed.json");
const config = require("../../settings/config.json");
const moment = require("moment");
const schema = require("../../Models/code");
const User = require("../../Models/User");

module.exports = new Command({
  // options
  name: "redeem",
  description: `redeem preium code`,
  userPermissions: [],
  category: "Premium",
  options: [
    {
      name: "code",
      description: `give me code`,
      type: "STRING",
      required: true,
    },
  ],
  // command start
  run: async ({ client, interaction, args }) => {
    // Code
    // Check if the user with a unique ID is in our database.
    let user = await User.findOne({
      Id: interaction.user.id, // if you are using slash commands, swap message with interaction.
    });

    // Check Users input for a valid code. Like `!redeem ABCD-EFGH-IJKL`
    let code = interaction.options.getString("code");

    // Return an error if the User does not include any Premium Code
    if (!code) {
      interaction.followUp(`**Please specify the code you want to redeem!**`);
    }

    // If the user is already a premium user, we dont want to save that so we return it.
    if (user && user.isPremium) {
      return interaction.followUp(`**> You already are a premium user**`);
    }

    // Check if the code is valid within the database
    const premium = await schema.findOne({
      code: code.toUpperCase(),
    });

    // Set the expire date for the premium code
    if (premium) {
      const expires = moment(premium.expiresAt).format(
        "dddd, MMMM Do YYYY HH:mm:ss"
      );

      // Once the code is expired, we delete it from the database and from the users profile
      user.isPremium = true;
      user.premium.redeemedBy.push(interaction.user);
      user.premium.redeemedAt = Date.now();
      user.premium.expiresAt = premium.expiresAt;
      user.premium.plan = premium.plan;

      // Save the User within the Database
      user = await user.save({ new: true }).catch(() => {});
      client.userSettings.set(interaction.user.id, user);
      await premium.deleteOne().catch(() => {});

      // Send a success message once redeemed
      interaction.followUp(
        `**You have successfully redeemed premium!**\n\n\`Expires at: ${expires}\``
      );

      // Error message if the code is not valid.
    } else {
      return interaction.followUp(
        `**The code is invalid. Please try again using valid one!**`
      );
    }
  },
});
```

if you want to make premium users list command follow this code

```js
const { Command } = require("reconlx");
const ee = require("../../settings/embed.json");
const config = require("../../settings/config.json");
const User = require("../../Models/User");
const moment = require("moment");
const { Collection, MessageEmbed } = require("discord.js");

module.exports = new Command({
  // options
  name: "premiumlist",
  description: `get list of all preimium user`,
  userPermissions: [],
  category: "Premium",
  // command start
  run: async ({ client, interaction, args }) => {
    // Code
    if (interaction.user.id !== "882481863661342770")
      return interaction.followUp(`You are not my Owner`);

    let data = client.userSettings
      .filter((data) => data.isPremium === true)
      .map((data, index) => {
        return ` <@${data.Id}> Expire At :- \`${moment(
          data.premium.expiresAt
        ).format("dddd, MMMM Do YYYY")}\` Plan :- \`${data.premium.plan}\` `;
      });
    interaction.followUp({
      embeds: [
        new MessageEmbed().setDescription(
          data.join("\n") || "No Premium User Found"
        ),
      ],
    });
  },
});

```

if you want to remove a user from premium list follow this code

```js
const { Command } = require("reconlx");
const ee = require("../../settings/embed.json");
const config = require("../../settings/config.json");
const moment = require("moment");
const schema = require("../../Models/code");
const User = require("../../Models/User");

module.exports = new Command({
  // options
  name: "rempremium",
  description: `remove preium from user`,
  userPermissions: [],
  category: "Premium",
  options: [
    {
      name: "user",
      description: `mention a premium user`,
      type: "USER",
      required: true,
    },
  ],
  // command start
  run: async ({ client, interaction, args }) => {
    // Code
    let user = interaction.options.getUser("user");
    let data = client.userSettings.get(user.id);
    if (!data.isPremium) {
      return interaction.followUp(`${user} is Not a Premium User`);
    } else {
      await User.findOneAndRemove({ Id: user.id });
      await client.userSettings.delete(user.id);
      interaction.followUp(`${user} Removed From Premium`);
    }
  },
});

```


Perfect. But as you can see, we have a Collection called `userSettings` within our Code.
Lets add it real quick to our Project before we are trying out our new Commands.

Go into your main file `index.js` / `main.js` / `app.js` ..
In there, define `client` with the neccessary intents.
Once done, add the following line into your code. \*_It should be below the client defination!!_

```js
client.userSettings = new Collection();
```

**OPTIONAL**
The next step requires some working brain, try understanding it first before trying it.
Go into your `interactionCreate.js` file. Define our Schema `User` and add it into the `command`.
You can also use your own and just integrate the User into it.
It's necessary so the bot can separate premium users of normal users.
