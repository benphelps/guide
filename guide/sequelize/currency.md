# Making a Currency System

A common feature of Discord bots is a currency system. It's possible to do everything in one object, but we can also abstract that in terms of *relations* between objects. This is where the power of a RDBMS (Relational Database Management System) truly shines. Sequelize calls these *associations*, so we'll be using that term from now on.

## File overview

There will be multiple files: a DB init script, your models, and your bot script. In [the previous Sequelize guide](/sequelize/), we placed all of these in the same file. Having everything in one file isn't an ideal practice, so we'll correct that.

This time we'll have six files.

* `app.js` is where we'll keep the main bot code.
* `dbInit.js` is the initialization file for the database. We run this once and forget about it.
* `dbObjects.js` is where we'll import the models and create associations here.
* `models/Users.js` is the Users model. Users will have a currency attribute in here.
* `models/CurrencyShop.js` is the Shop model. The shop will have a name and a price for each item.
* `models/UserItems.js` is the junction table between the users and the shop. A junction table connects two tables. Our junction table will have an additional field for the amount of that item the user has.

## Create models

Here is an entity relation diagram of the models we'll be making:

![Currency database structure diagram](./images/currency_er_diagram.svg)

`Users` have a `user_id`, and a `balance`. Each `user_id` can have multiple links to the `UserItems` table, and each entry in the table connects to one of the items in the `CurrencyShop`, which will have a `name` and a `cost` associated with it.

To implement this, begin by making a `models` folder and create a `Users.js` file inside which contains the following:

```js
module.exports = (sequelize, DataTypes) => {
	return sequelize.define('users', {
		user_id: {
			type: DataTypes.STRING,
			primaryKey: true,
		},
		balance: {
			type: DataTypes.INTEGER,
			defaultValue: 0,
			allowNull: false,
		},
	}, {
		timestamps: false,
	});
};
```

Like you see in the diagram above, the Users model will only have two attributes: a `user_id` primary key and a `balance`. A primary key is a particular attribute that becomes the default column used when joining tables together, and it is automatically unique and not `null`.

Balance also sets `allowNull` to `false`, which means that both values have to be set in conjunction with creating a primary key; otherwise, the database would throw an error. This constraint guarantees correctness in your data storage. You'll never have `null` or empty values, ensuring that if you somehow forget to validate in the application that both values are not `null`, the database would do a final validation.

Notice that the options object sets `timestamps` to `false`. This option disables the `createdAt` and the `updatedAt` columns that sequelize usually creates for you. Setting `user_id` to primary also eliminates the `id` primary key that Sequelize usually generates for you since there can only be one primary key on a table.

Next, still in the same `models` folder, create a `CurrencyShop.js` file that contains the following:

```js
module.exports = (sequelize, DataTypes) => {
	return sequelize.define('currency_shop', {
		name: {
			type: DataTypes.STRING,
			unique: true,
		},
		cost: {
			type: DataTypes.INTEGER,
			allowNull: false,
		},
	}, {
		timestamps: false,
	});
};
```

Like the Users model, timestamps aren't needed here, so you can disable it. Unlike the Users model, however, the `unique` field is set to `true` here, allowing you to change the name without affecting the primary key that joins this to the next object. This gets generated automatically by sequelize since a primary key isn't set.

The next file will be `UserItems.js`, the junction table.

```js
module.exports = (sequelize, DataTypes) => {
	return sequelize.define('user_item', {
		user_id: DataTypes.STRING,
		item_id: DataTypes.INTEGER,
		amount: {
			type: DataTypes.INTEGER,
			allowNull: false,
			'default': 0,
		},
	}, {
		timestamps: false,
	});
};
```

The junction table will link `user_id` and the `id` of the currency shop together. It also contains an `amount` number, which indicates how many of that item a user has.

## Initialize database

Now that the models are defined, you should create them in your database to access them in the bot file. We ran the sync inside the `ready` event in the previous tutorial, which is entirely unnecessary since it only needs to run once. You can make a file to initialize the database and never touch it again unless you want to remake the entire database.

Create a file called `dbInit.js` in the base directory (*not* in the `models` folder).

::: danger
Make sure you use version 5 or later of Sequelize! Version 4, as used in this guide, will pose a security threat. You can read more about this issue on the [Sequelize issue tracker](https://github.com/sequelize/sequelize/issues/7310).
:::

```js
const Sequelize = require('sequelize');

const sequelize = new Sequelize('database', 'username', 'password', {
	host: 'localhost',
	dialect: 'sqlite',
	logging: false,
	storage: 'database.sqlite',
});

const CurrencyShop = require('./models/CurrencyShop.js')(sequelize, Sequelize.DataTypes);
require('./models/Users.js')(sequelize, Sequelize.DataTypes);
require('./models/UserItems.js')(sequelize, Sequelize.DataTypes);

const force = process.argv.includes('--force') || process.argv.includes('-f');

sequelize.sync({ force }).then(async () => {
	const shop = [
		CurrencyShop.upsert({ name: 'Tea', cost: 1 }),
		CurrencyShop.upsert({ name: 'Coffee', cost: 2 }),
		CurrencyShop.upsert({ name: 'Cake', cost: 5 }),
	];

	await Promise.all(shop);
	console.log('Database synced');

	sequelize.close();
}).catch(console.error);
```

Here you pull the two models and the junction table from the respective model declarations, sync them, and add items to the shop.

A new function here is the `.upsert()` function. It's a portmanteau for **up**date or in**sert**. `upsert` is used here to avoid creating duplicates if you run this file multiple times. That shouldn't happen because `name` is defined as *unique*, but there's no harm in being safe. Upsert also has a nice side benefit: if you adjust the cost, the respective item should also have their cost updated.

::: tip
Execute `node dbInit.js` to create the database tables. Unless you make a change to the models, you'll never need to touch the file again. If you change a model, you can execute `node dbInit.js --force` or `node dbInit.js -f` to force sync your tables. It's important to note that this **will** empty and remake your model tables.
:::

## Create associations

Next, add the associations to the models. Create a file named `dbObjects.js` in the base directory, next to `dbInit.js`.

```js
const Sequelize = require('sequelize');

const sequelize = new Sequelize('database', 'username', 'password', {
	host: 'localhost',
	dialect: 'sqlite',
	logging: false,
	storage: 'database.sqlite',
});

const Users = require('./models/Users.js')(sequelize, Sequelize.DataTypes);
const CurrencyShop = require('./models/CurrencyShop.js')(sequelize, Sequelize.DataTypes);
const UserItems = require('./models/UserItems.js')(sequelize, Sequelize.DataTypes);

UserItems.belongsTo(CurrencyShop, { foreignKey: 'item_id', as: 'item' });

Reflect.defineProperty(Users.prototype, 'addItem', {
	value: async item => {
		const userItem = await UserItems.findOne({
			where: { user_id: this.user_id, item_id: item.id },
		});

		if (userItem) {
			userItem.amount += 1;
			return userItem.save();
		}

		return UserItems.create({ user_id: this.user_id, item_id: item.id, amount: 1 });
	},
});

Reflect.defineProperty(Users.prototype, 'getItems', {
	value: () => {
		return UserItems.findAll({
			where: { user_id: this.user_id },
			include: ['item'],
		});
	},
});

module.exports = { Users, CurrencyShop, UserItems };
```

Note that the connection object could be abstracted in another file and had both `dbInit.js` and `dbObjects.js` use that connection file, but it's not necessary to overly abstract things.

Another new method here is the `.belongsTo()` method. Using this method, you add `CurrencyShop` as a property of `UserItem` so that when you do `userItem.item`, you get the respectively attached item. You use `item_id` as the foreign key so that it knows which item to reference.

You then add some methods to the `Users` object to finish up the junction: add items to users, and get their current inventory. The code inside should be somewhat familiar from the last tutorial. `.findOne()` is used to get the item if it exists in the user's inventory. If it does, increment it; otherwise, create it.

Getting items is similar; use `.findAll()` with the user's id as the key. The `include` key is for associating the CurrencyShop with the item. You must explicitly tell Sequelize to honor the `.belongsTo()` association; otherwise, it will take the path of the least effort.

## Application code

Create an `app.js` file in the base directory with the following skeleton code to put it together.

<!-- eslint-disable require-await -->

```js
const { Op } = require('sequelize');
const { Client, codeBlock, Collection, GatewayIntentBits } = require('discord.js');
const { Users, CurrencyShop } = require('./dbObjects.js');

const client = new Client({ intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMessages] });
const currency = new Collection();

client.once('ready', async () => {
	console.log(`Logged in as ${client.user.tag}!`);
});

client.on('messageCreate', async message => {
	if (message.author.bot) return;
	currency.add(message.author.id, 1);
});

client.on('interactionCreate', async interaction => {
	if (!interaction.isChatInputCommand()) return;

	const { commandName } = interaction;
	// ...
});

client.login('your-token-goes-here');
```

Nothing special about this skeleton. You import the Users and CurrencyShop models from our `dbObjects.js` file and add a currency Collection. Every time someone talks, add 1 to their currency count. The rest is just standard discord.js code and a simple if/else command handler. A Collection is used for the `currency` variable to cache individual users' currency, so you don't have to hit the database for every lookup. An if/else handler is used here, but you can put it in a framework or command handler as long as you maintain a reference to the models and the currency collection.

### Helper methods

```js {4-25}
const client = new Client({ intents: [GatewayIntentBits.Guilds, GatewayIntentBits.GuildMessages] });
const currency = new Collection();

Reflect.defineProperty(currency, 'add', {
	value: async (id, amount) => {
		const user = currency.get(id);

		if (user) {
			user.balance += Number(amount);
			return user.save();
		}

		const newUser = await Users.create({ user_id: id, balance: amount });
		currency.set(id, newUser);

		return newUser;
	},
});

Reflect.defineProperty(currency, 'getBalance', {
	value: id => {
		const user = currency.get(id);
		return user ? user.balance : 0;
	},
});
```

This defines an `.add()` method to our currency collection. You'll use it quite frequently, so having a method for it makes your life easier. A `.getBalance()` method is also defined, to ensure that a number is always returned.

### Ready event data sync

```js {2-3}
client.once('ready', async () => {
	const storedBalances = await Users.findAll();
	storedBalances.forEach(b => currency.set(b.user_id, b));

	console.log(`Logged in as ${client.user.tag}!`);
});
```

In the ready event, sync the currency collection with the database for easy access later.

### Show user balance

```js {7-9}
client.on('interactionCreate', async interaction => {
	if (!interaction.isChatInputCommand()) return;

	const { commandName } = interaction;

	if (commandName === 'balance') {
		const target = interaction.options.getUser('user') ?? interaction.user;

		return interaction.reply(`${target.tag} has ${currency.getBalance(target.id)}💰`);
	}
});
```

Nothing tricky here. The `.getBalance()` method is used to show either the author's or the mentioned user's balance.

### Show user inventory

<!-- eslint-skip -->

```js {5-11}
if (commandName === 'balance') {
	// ...
}
else if (commandName === 'inventory') {
	const target = interaction.options.getUser('user') ?? interaction.user;
	const user = await Users.findOne({ where: { user_id: target.id } });
	const items = await user.getItems();

	if (!items.length) return interaction.reply(`${target.tag} has nothing!`);

	return interaction.reply(`${target.tag} currently has ${items.map(i => `${i.amount} ${i.item.name}`).join(', ')}`);
}
```
This is where you begin to see the power of associations. Even though users and the shop are different tables, and the data is stored separately, you can get a user's inventory by looking at the junction table and join it with the shop; no duplicated item names that waste space!

### Transfer currency to another user

<!-- eslint-skip -->

```js {2-12}
else if (commandName === 'transfer') {
	const currentAmount = currency.getBalance(interaction.user.id);
	const transferAmount = interaction.options.getInteger('amount');
	const transferTarget = interaction.options.getUser('user');

	if (transferAmount > currentAmount) return interaction.reply(`Sorry ${interaction.user}, you only have ${currentAmount}.`);
	if (transferAmount <= 0) return interaction.reply(`Please enter an amount greater than zero, ${interaction.user}.`);

	currency.add(interaction.user.id, -transferAmount);
	currency.add(transferTarget.id, transferAmount);

	return interaction.reply(`Successfully transferred ${transferAmount}💰 to ${transferTarget.tag}. Your current balance is ${currency.getBalance(interaction.user.id)}💰`);
}
```
As a bot creator, you should always be thinking about how to make the user experience better. Good UX makes users less frustrated with your commands. If your inputs are different types, don't make them memorize which parameters come before the other.

You'd ideally want to allow users to do both `!transfer 5 @user` and `!transfer @user 5`. To get the amount, you can grab the first non-mention text in the command. In the second line of the above code: split the command by spaces and look for anything that doesn't match a mention; you can assume that's the transfer amount. Then do some checking to make sure it's a valid input. You can also do error checking on the transfer target, but we won't include that here because of its triviality.

`.add()` is used for both removing and adding currency. Since transfer amounts below zero are disallowed, it's safe to apply the transfer amount's additive inverse to their balance.

### Buying an item

<!-- eslint-skip -->

```js {2-14}
else if (commandName === 'buy') {
	const itemName = interaction.options.getString('item');
	const item = await CurrencyShop.findOne({ where: { name: { [Op.like]: itemName } } });

	if (!item) return interaction.reply(`That item doesn't exist.`);
	if (item.cost > currency.getBalance(interaction.user.id)) {
		return interaction.reply(`You currently have ${currency.getBalance(interaction.user.id)}, but the ${item.name} costs ${item.cost}!`);
	}

	const user = await Users.findOne({ where: { user_id: interaction.user.id } });
	currency.add(interaction.user.id, -item.cost);
	await user.addItem(item);

	return interaction.reply(`You've bought: ${item.name}.`);
}
```

For users to search for an item without caring about the letter casing, you can use the `$iLike` modifier when looking for the name. Keep in mind that this may be slow if you have millions of items, so please don't put a million items in your shop.

### Display the shop

<!-- eslint-skip -->

```js {2-3}
else if (commandName === 'shop') {
	const items = await CurrencyShop.findAll();
	return interaction.reply(codeBlock(items.map(i => `${i.name}: ${i.cost}💰`).join('\n')));
}
```
There's nothing special here; just a regular `.findAll()` to get all the items in the shop and `.map()` to transform that data into something nice looking.

### Display the leaderboard

<!-- eslint-skip -->

```js {2-10}
else if (commandName === 'leaderboard') {
	return interaction.reply(
		codeBlock(
			currency.sort((a, b) => b.balance - a.balance)
				.filter(user => client.users.cache.has(user.user_id))
				.first(10)
				.map((user, position) => `(${position + 1}) ${(client.users.cache.get(user.user_id).tag)}: ${user.balance}💰`)
				.join('\n'),
		),
	);
}
```

Nothing extraordinary here either. You could query the database for the top ten currency holders, but since you already have access to them locally inside the `currency` variable, you can sort the Collection and use `.map()` to display it in a friendly format. The filter is in case the users no longer exist in the bot's cache.

## Resulting code

<ResultingCode />
