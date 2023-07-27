---
title: 'Writing your first plugin'
sidebar_position: 4
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';

In Vendure **plugins** are used to extend the core functionality of the server. Plugins can be pre-made functionality that you can install via npm, or they can be custom plugins that you write yourself.

For any unit of functionality that you need to add to your project, you'll be writing creating a Vendure plugin. By convention, plugins are stored in the `plugins` directory of your project. However, this is not a requirement, and you are free to arrange your plugin files in any way you like.

```txt
├──src
    ├── index.ts
    ├── vendure-config.ts
    ├── plugins
        ├── reviews-plugin
        ├── cms-plugin
        ├── wishlist-plugin
        ├── stock-sync-plugin
```

:::info
For a complete working example of a Vendure plugin, see the [real-world-vendure Reviews plugin](https://github.com/vendure-ecommerce/real-world-vendure/tree/master/src/plugins/reviews)

If you intend to write a shared plugin to be distributed as an npm package, see the [vendure plugin-template repo](https://github.com/vendure-ecommerce/plugin-template)
:::

In this guide, we will implement a simple but fully-functional **wishlist plugin** step-by-step. The goal of this plugin is to allow signed-in customers to add products to a wishlist, and to view and manage their wishlist.

## Step 1: Create the plugin file

We'll start by creating a new directory to house our plugin, add create the main plugin file:

```txt
├──src
    ├── index.ts
    ├── vendure-config.ts
    ├── plugins
        // highlight-next-line
        ├── wishlist-plugin
            // highlight-next-line
            ├── wishlist.plugin.ts
```

```ts title="src/plugins/reviews-plugin/reviews.plugin.ts"
import { PluginCommonModule, VendurePlugin } from '@vendure/core';

@VendurePlugin({
    imports: [PluginCommonModule],
})
export class WishlistPlugin {}
```

The `PluginCommonModule` will be required in all plugins that you create. It contains the common services that are exposed by Vendure Core, allowing you to inject them into your plugin's services and resolvers.

## Step 2: Define an entity

Next we will define a new database entity to store the wishlist items. Vendure uses [TypeORM](https://typeorm.io/) to manage the database schema, and an Entity corresponds to a database table.

First let's create the file to house the entity:

```txt
├── wishlist-plugin
    ├── wishlist.plugin.ts
    ├── entities
        // highlight-next-line
        ├── wishlist-item.entity.ts
```

By convention, we'll store the entity definitions in the `entities` directory of the plugin. Again, this is not a requirement, but it is a good way to keep your plugin organized.

```ts title="src/plugins/wishlist-plugin/entities/wishlist-item.entity.ts"
import { DeepPartial, ID, ProductVariant, VendureEntity } from '@vendure/core';
import { Column, Entity, ManyToOne } from 'typeorm';

@Entity()
export class WishlistItem extends VendureEntity {
    constructor(input?: DeepPartial<WishlistItem>) {
        super(input);
    }

    @ManyToOne(type => ProductVariant)
    productVariant: ProductVariant;

    @Column()
    productVariantId: ID;
}
```

Let's break down what's happening here:

-   The `WishlistItem` entity extends the [`VendureEntity` class](/reference/typescript-api/entities/vendure-entity/). This is a base class which provides the `id`, `createdAt` and `updatedAt` fields, and all custom entities should extend it.
-   The `@Entity()` decorator marks this class as a TypeORM entity.
-   The `@ManyToOne()` decorator defines a many-to-one relationship with the `ProductVariant` entity. This means that each `WishlistItem` will be associated with a single `ProductVariant`.
-   The `productVariantId` column is not strictly necessary, but it allows us to always have access to the ID of the related `ProductVariant` without having to load the entire `ProductVariant` entity from the database.
-   The `constructor()` is used to create a new instance of the entity. This is not strictly necessary, but it is a good practice to define a constructor which takes a `DeepPartial` of the entity as an argument. This allows us to create new instances of the entity using the `new` keyword, passing in a plain object with the desired properties.

Next we need to register this entity with our plugin:

```ts title="src/plugins/wishlist-plugin/wishlist.plugin.ts"
import { PluginCommonModule, VendurePlugin } from '@vendure/core';
import { WishlistItem } from './entities/wishlist-item.entity';

@VendurePlugin({
    imports: [PluginCommonModule],
    entities: [WishlistItem],
})
export class WishlistPlugin {}
```

## Step 3: Add a custom field to the Customer entity

We'll now define a new custom field on the Customer entity which will store a list of WishlistItems. This will allow us to easily query for all wishlist items associated with a particular customer.

Custom fields are defined in the VendureConfig object, and in a plugin we use the `configuration` function to modify the config object:

```ts title="src/plugins/wishlist-plugin/wishlist.plugin.ts"
import { PluginCommonModule, VendurePlugin } from '@vendure/core';
import { WishlistItem } from './entities/wishlist-item.entity';

@VendurePlugin({
    imports: [PluginCommonModule],
    entities: [WishlistItem],
    configuration: config => {
        config.customFields.Customer.push({
            name: 'wishlistItems',
            type: 'relation',
            list: true,
            entity: WishlistItem,
            internal: true,
        });
        return config;
    },
})
export class WishlistPlugin {}
```

In this snippet we are pushing a new custom field definition onto the `Customer` entity's `customFields` array, and defining this new field as a list (array) of `WishlistItem` entities. Internally, this will tell TypeORM to update the database schema to store this new field. We set `internal: true` to indicate that this field should not be directly exposed to the GraphQL API as `Customer.customFields.wishlistItems`, but instead should be accessed via a custom resolver we will define later.

In order to make use of this custom field in a type-safe way, we can tell TypeScript about this field in a new file:

```txt
├── wishlist-plugin
    ├── wishlist.plugin.ts
    // highlight-next-line
    ├── types.ts
```

```ts title="src/plugins/wishlist-plugin/types.ts"
import { CustomCustomerFields } from '@vendure/core/dist/entity/custom-entity-fields';
import { WishlistItem } from './entities/wishlist-item.entity';

declare module '@vendure/core/dist/entity/custom-entity-fields' {
  interface CustomCustomerFields {
    wishlistItems: WishlistItem[];
  }
}
```

We can then import this types file in our plugin's main file:

```ts title="src/plugins/wishlist-plugin/wishlist.plugin.ts"
// highlight-next-line
import './types';
```

## Step 4: Create a service

A "service" is a class which houses the bulk of the business logic of any plugin. A plugin can define multiple services if needed, but each service should be responsible for a single unit of functionality, such as dealing with a particular entity, or performing a particular task.

Let's create a service to handle the wishlist functionality:

```txt
├── wishlist-plugin
    ├── wishlist.plugin.ts
    ├── services
        // highlight-next-line
        ├── wishlist.service.ts
```

```ts title="src/plugins/wishlist-plugin/wishlist.service.ts"
import { Injectable } from '@nestjs/common';
import {
    Customer,
    ForbiddenError,
    ID,
    InternalServerError,
    ProductVariantService,
    RequestContext,
    TransactionalConnection,
    UserInputError,
} from '@vendure/core';

import { WishlistItem } from '../entities/wishlist-item.entity';

@Injectable()
export class WishlistService {
    constructor(
        private connection: TransactionalConnection,
        private productVariantService: ProductVariantService,
    ) {}

    async getWishlistItems(ctx: RequestContext): Promise<WishlistItem[]> {
        try {
            const customer = await this.getCustomerWithWishlistItems(ctx);
            return customer.customFields.wishlistItems;
        } catch (err: any) {
            return [];
        }
    }

    /**
     * Adds a new item to the active Customer's wishlist.
     */
    async addItem(ctx: RequestContext, variantId: ID): Promise<WishlistItem[]> {
        const customer = await this.getCustomerWithWishlistItems(ctx);
        const variant = this.productVariantService.findOne(ctx, variantId);
        if (!variant) {
            throw new UserInputError(`No ProductVariant with the id ${variantId} could be found`);
        }
        const existingItem = customer.customFields.wishlistItems.find(i => i.productVariantId === variantId);
        if (existingItem) {
            // Item already exists in wishlist, do not
            // add it again
            return customer.customFields.wishlistItems;
        }
        const wishlistItem = await this.connection
            .getRepository(ctx, WishlistItem)
            .save(new WishlistItem({ productVariantId: variantId }));
        customer.customFields.wishlistItems.push(wishlistItem);
        await this.connection.getRepository(ctx, Customer).save(customer, { reload: false });
        return this.getWishlistItems(ctx);
    }

    /**
     * Removes an item from the active Customer's wishlist.
     */
    async removeItem(ctx: RequestContext, itemId: ID): Promise<WishlistItem[]> {
        const customer = await this.getCustomerWithWishlistItems(ctx);
        const itemToRemove = customer.customFields.wishlistItems.find(i => i.id === itemId);
        if (itemToRemove) {
            await this.connection.getRepository(ctx, WishlistItem).remove(itemToRemove);
            customer.customFields.wishlistItems = customer.customFields.wishlistItems.filter(
                i => i.id !== itemId,
            );
        }
        await this.connection.getRepository(ctx, Customer).save(customer);
        return this.getWishlistItems(ctx);
    }

    /**
     * Gets the active Customer from the context and loads the wishlist items.
     */
    private async getCustomerWithWishlistItems(ctx: RequestContext): Promise<Customer> {
        if (!ctx.activeUserId) {
            throw new ForbiddenError();
        }
        const customer = await this.connection.getRepository(ctx, Customer).findOne({
            where: { user: { id: ctx.activeUserId } },
            relations: {
                customFields: {
                    wishlistItems: {
                        productVariant: true,
                    },
                },
            },
        });
        if (!customer) {
            throw new InternalServerError(`Customer was not found`);
        }
        return customer;
    }
}
```

Let's break down what's happening here:

- The `WishlistService` class is decorated with the `@Injectable()` decorator. This is a standard NestJS decorator which tells the NestJS dependency injection (DI) system that this class can be injected into other classes. All your services should be decorated with this decorator.
- The arguments passed to the constructor will be injected by the NestJS DI system. The `connection` argument is a [TransactionalConnection](/reference/typescript-api/data-access/transactional-connection/) instance, which is used to access and manipulate data in the database. The [`ProductVariantService`](/reference/typescript-api/services/product-variant-service/) argument is a built-in Vendure service which contains methods relating to ProductVariants.
- The [`RequestContext`](/reference/typescript-api/request/request-context/) object is usually the first argument to any service method, and contains information and context about the current request as well as any open database transactions. It should always be passed to the methods of the `TransactionalConnection`.

The service is then registered with the plugin metadata as a provider:

```ts title="src/plugins/wishlist-plugin/wishlist.plugin.ts"
import { PluginCommonModule, VendurePlugin } from '@vendure/core';
import { WishlistService } from './services/wishlist.service';

@VendurePlugin({
    imports: [PluginCommonModule],
    // highlight-next-line
    providers: [WishlistService],
    entities: [WishlistItem],
    configuration: config => {
        // ...
    },
})
export class WishlistPlugin {}
```

## Step 5: Extend the GraphQL API

This plugin will need to extend the Shop API, adding new mutations and queries to enable the customer to view and manage their wishlist.

First we will create a new file to hold the GraphQL schema extensions:

```txt
├── wishlist-plugin
    ├── wishlist.plugin.ts
    ├── api
        // highlight-next-line
        ├── api-extensions.ts
```

```ts title="src/plugins/wishlist-plugin/api/api-extensions.ts"
import gql from 'graphql-tag';

export const shopApiExtensions = gql`
    type WishlistItem implements Node {
        id: ID!
        createdAt: DateTime!
        updatedAt: DateTime!
        productVariant: ProductVariant!
        productVariantId: ID!
    }

    extend type Query {
        activeCustomerWishlist: [WishlistItem!]!
    }

    extend type Mutation {
        addToWishlist(productVariantId: ID!): [WishlistItem!]!
        removeFromWishlist(itemId: ID!): [WishlistItem!]!
    }
`;
```

:::note

The `graphql-tag` package is a dependency of the Vendure core package. Depending on the package manager you are using, you may need to install it separately with `yarn add graphql-tag` or `npm install graphql-tag`.

:::

The `api-extensions.ts` file is where we define the extensions we will be making to the Shop API GraphQL schema. We are defining a new `WishlistItem` type; a new query: `activeCustomerWishlist`; and two new mutations: `addToWishlist` and `removeFromWishlist`. This definition is written in [schema definition language](https://graphql.org/learn/schema/) (SDL), a convenient syntax for defining GraphQL schemas.

Next we need to pass these extensions to our plugin's metadata:

```ts title="src/plugins/wishlist-plugin/wishlist.plugin.ts"
import { PluginCommonModule, VendurePlugin } from '@vendure/core';
import { shopApiExtensions } from './api/api-extensions';

@VendurePlugin({
    imports: [PluginCommonModule],
    shopApiExtensions: {
        schema: shopApiExtensions,
        resolvers: [],
    },
})
export class WishlistPlugin {}
```

## Step 6: Create a resolver

Now that we have defined the GraphQL schema extensions, we need to create a resolver to handle the new queries and mutations. A resolver in GraphQL is a function which actually implements the query or mutation defined in the schema. This is done by creating a new file in the `api` directory:

```txt
├── wishlist-plugin
    ├── wishlist.plugin.ts
    ├── api
        ├── api-extensions.ts
        // highlight-next-line
        ├── wishlist.resolver.ts
```

```ts title="src/plugins/wishlist-plugin/api/wishlist.resolver.ts"
import { Args, Mutation, Query, Resolver } from '@nestjs/graphql';
import { Allow, Ctx, Permission, RequestContext, Transaction } from '@vendure/core';

import { WishlistItem } from '../entities/wishlist-item.entity';
import { WishlistService } from '../service/wishlist.service';

@Resolver()
export class WishlistShopResolver {
    constructor(private wishlistService: WishlistService) {}

    @Query()
    @Allow(Permission.Owner)
    activeCustomerWishlist(@Ctx() ctx: RequestContext) {
        return this.wishlistService.getWishlistItems(ctx);
    }

    @Mutation()
    @Transaction()
    @Allow(Permission.Owner)
    async addToWishlist(
        @Ctx() ctx: RequestContext,
        @Args() { productVariantId }: { productVariantId: string },
    ) {
        return this.wishlistService.addItem(ctx, productVariantId);
    }

    @Mutation()
    @Transaction()
    @Allow(Permission.Owner)
    async removeFromWishlist(@Ctx() ctx: RequestContext, @Args() { itemId }: { itemId: string }) {
        return this.wishlistService.removeItem(ctx, itemId);
    }
}
```

Resolvers are usually "thin" functions that delegate the actual work to a service. Vendure, like NestJS itself, makes heavy use of decorators at the API layer to define various aspects of the resolver. Let's break down what's happening here:

- The `@Resolver()` decorator tells the NestJS DI system that this class is a resolver. Since a Resolver is part of the NestJS DI system, we can also inject dependencies into its constructor. In this case we are injecting the `WishlistService` which we created in the previous step.
- The `@Mutation()` decorator tells Vendure that this is a mutation resolver. Similarly, `@Query()` decorator defines a query resolver. The name of the method is the name of the query or mutation in the schema.
- The `@Transaction()` decorator tells Vendure that this resolver method should be wrapped in a database transaction. This is important because we are performing multiple database operations in this method, and we want them to be atomic.
- The `@Allow()` decorator tells Vendure that this mutation is only allowed for users with the `Owner` permission. The `Owner` permission is a special permission which indicates that the active user should be the owner of this operation. 
- The `@Ctx()` decorator tells Vendure that this method requires access to the `RequestContext` object. Every resolver should have this as the first argument, as it is required throughout the Vendure request lifecycle.

This resolver is then registered with the plugin metadata:

```ts title="src/plugins/wishlist-plugin/wishlist.plugin.ts"
import { PluginCommonModule, VendurePlugin } from '@vendure/core';
import { shopApiExtensions } from './api/api-extensions';
import { WishlistShopResolver } from './api/wishlist.resolver';

@VendurePlugin({
    imports: [PluginCommonModule],
    shopApiExtensions: {
        schema: shopApiExtensions,
        // highlight-next-line
        resolvers: [WishlistShopResolver],
    },
    configuration: config => {
        // ...
    },
})
export class WishlistPlugin {}
```

:::info

More information about resolvers can be found in the [NestJS docs](https://docs.nestjs.com/graphql/resolvers).

:::

## Step 7: Specify compatibility

Since Vendure v2.0.0, it is possible for a plugin to specify which versions of Vendure core it is compatible with. This is especially important if the plugin is intended to be made publicly available via npm or another package registry.

The compatibility is specified via the `compatibility` property in the plugin metadata:

```ts title="src/plugins/wishlist-plugin/wishlist.plugin.ts"
@VendurePlugin({
    // ...
    // highlight-next-line
    compatibility: '^2.0.0',
})
export class WishlistPlugin {}
```

The value of this property is a [semver range](https://docs.npmjs.com/about-semantic-versioning) which specifies the range of compatible versions. In this case, we are saying that this plugin is compatible with any version of Vendure core which is `>= 2.0.0 < 3.0.0`.

## Step 8: Add the plugin to the VendureConfig

The final step is to add the plugin to the `VendureConfig` object. This is done in the `vendure-config.ts` file:

```ts title="src/vendure-config.ts"
import { VendureConfig } from '@vendure/core';
import { WishlistPlugin } from './plugins/wishlist-plugin/wishlist.plugin';

export const config: VendureConfig = {
    // ...
    plugins: [
        // ...
        // highlight-next-line
        WishlistPlugin,
    ],
};
```

## Test the plugin

Now that the plugin is installed, we can test it out. Since we have defined a custom field, we'll need to generate and run a migration to add the new column to the database:

```bash
npm run migration:generate wishlist-plugin
```

Then start the server:

```bash
npm run dev
```

Once the server is running, we should be able to log in as an existing Customer, and then add a product to the wishlist:

<Tabs>
<TabItem value="Login mutation" label="Login mutation" default>

```graphql
mutation Login {
    login(username: "alec.breitenberg@gmail.com", password: "test") {
        ... on CurrentUser {
            id
            identifier
        }
        ... on ErrorResult {
            errorCode
            message
        }
    }
}
```

</TabItem>
<TabItem value="Response" label="Response">

```json
{
  "data": {
    "login": {
      "id": "9",
      "identifier": "alec.breitenberg@gmail.com"
    }
  }
}
```

  </TabItem>
</Tabs>


<Tabs>
<TabItem value="AddToWishlist mutation" label="AddToWishlist mutation" default>

```graphql
mutation AddToWishlist {
    addToWishlist(productVariantId: "7") {
        id
        productVariant {
            id
            name
        }
    }
}
```

</TabItem>
<TabItem value="Response" label="Response">

```json
{
  "data": {
    "addToWishlist": [
      {
        "id": "4",
        "productVariant": {
          "id": "7",
          "name": "Wireless Optical Mouse"
        }
      }
    ]
  }
}
```

  </TabItem>
</Tabs>

We can then query the wishlist items:


<Tabs>
<TabItem value="GetWishlist mutation" label="GetWishlist mutation" default>

```graphql
query GetWishlist {
    activeCustomerWishlist {
        id
        productVariant {
            id
            name
        }
    }
}
```

</TabItem>
<TabItem value="Response" label="Response">

```json
{
  "data": {
    "activeCustomerWishlist": [
      {
        "id": "4",
        "productVariant": {
          "id": "7",
          "name": "Wireless Optical Mouse"
        }
      }
    ]
  }
}
```

  </TabItem>
</Tabs>

And finally, we can test removing an item from the wishlist:

<Tabs>
<TabItem value="RemoveFromWishlist mutation" label="RemoveFromWishlist mutation" default>

```graphql
mutation RemoveFromWishlist {
    removeFromWishlist(itemId: "4") {
        id
        productVariant {
            name
        }
    }
}
```

</TabItem>
<TabItem value="Response" label="Response">

```json
{
  "data": {
    "removeFromWishlist": []
  }
}
```

  </TabItem>
</Tabs>