+++
date = '2025-02-19T09:24:46+01:00'
draft = true
title = 'Rql+Nestjs+Prisma: Search Made Easy'
tags = ['nestjs', 'prisma', 'rql']
+++

When you build API, you will always come to the state, where you will have to search among data which exists within your system. There are already standards how to deal with GET query parameters, such as

- [FIQL](https://tools.ietf.org/html/draft-nottingham-atompub-fiql-00)
- [RQL](https://dundalek.com/rql/draft-zyp-rql-00.html)

We chose RQL for our latest project, which is built upon [NestJS](https://nestjs.com/) framework and using [prisma](https://www.prisma.io/) ORM as database abstraction.

## What is Resource Query Language (RQL)?
RQL (Resource Query Language) is a query language designed for filtering and manipulating data in RESTful APIs. It provides a way to perform complex queries using a URL-friendly syntax, allowing operations like filtering, sorting, pagination, and aggregation. RQL is commonly used in APIs to enable flexible and efficient data retrieval without requiring custom query parameters for each use case. Read more about [RQL](https://www.sitepen.com/blog/resource-query-language-a-query-language-for-the-web-nosql).


Our goal is to receive URLs like `GET /contracts?q=eq(id,1234)` and translate to SQL `SELECT * FROM contacts WHERE id = 1234`.

## RqlService — map raw RQL into workable results
The goal of this service is to take a raw RQL statement and convert it to RqlResult . Do not be overwhelmed by the code underneath, I will explain each part as we go.

```typescript 
type OrderBy = 
  | { [x: string]: 'asc' | 'desc' }
  | { [x: string]: { [y: string]: 'asc' | 'desc' } };

export type RqlWhereType =
  | { [x: string]: { [y: string]: { [z: string]: null | number | string } } }
  | { [x: string]: { [y: string]: null | number | string } }
  | { [x: string]: null };
export type RqlWhereOption = { [x: string]: RqlWhereType[] } | RqlWhereType;

type RqlResult = {
  hardLimit: number;
  limit: number;
  offset: number;
  orderBy: OrderBy[];
  where: RqlWhereOption[];
}
```

### Either detour
The following example will use concept of Either and therefore it’s important to understand how it’s used. Either is functional programming error handling. It’s alternative to try/catch blocks. Either is a type that can contain only 2 possible values, either one or the other (no both!):

- An error value (called Left by convention)
- A successful return value (called Right by convention)

Another benefit of this approach is that you can “pipe” operations together, similary to linux pipe, provided that you stay on the same side.

Read more about Either: https://www.sandromaglione.com/articles/either-error-handling-functional-programming

### Context detour

Throughout the RqlService, we are going to work with Context.

```typescript
type Context = {
  hardLimit: number;
  limit: number;
  offset: number;
  orderBy: OrderBy[];
  rqls: RqlInstruction[];
  supportedColumns: string[];
  where: RqlWhereOption[];
};
```

We build this at the start, pass it any private function where it’s content will be modified accordingly.

#### Step 0: Parse function

RqlService has parse function which is responsible to take raw RQL statement and convert it to RqlResult . Let’s go though each step individually.

```typescript
parse(
  rql: string,
  supportedColumns: string[]
): Either<ContextAwareException, RqlResult> {
  const ctx: Context = {
    hardLimit: MAX_LIMIT,
    limit: 10,
    offset: 0,
    orderBy: [{ createdAt: 'asc' }],
    rqls: this.parseRQL(rql),
    supportedColumns,
    where: [],
  };

  return this.getLimitOffset(ctx)
    .bind((ctx) => this.getOrderBy(ctx))
    .bind((ctx) => this.getFilters(ctx))
    .bind((ctx) => this.toResponse(ctx));
}
```

#### Step 1: Parse raw RQL into instructions
Our goal is to transform raw RQL statement into an array of instructions we can work later with. RqlInstruction will hold information about operation which is requested and array of the args arguments, that comes with it.

```typescript
type RqlInstruction = {
  args: string[],
  operation: string
}

```
Ex. if we get `eq(id,1234)` would be transform into


```typescript
const instruction: RqlInstruction = {
  operation: 'eq',
  args: ['id', '1234']
}
```

```typescript
private parseRQL(rql: string): RqlInstruction[] {
  const groupPattern = /(\w+)?\[([^\[\]]+)\]/g;
  const groups = Array.from(rql.matchAll(groupPattern));
  const instrs: RqlInstruction[] = [];

  let rest = rql;

  groups.forEach(([raw, operation, args]) => {
    if (operation === 'or') {
      instrs.push({ args: [args], operation: 'or' });
    } else if (operation === 'not') {
      instrs.push({ args: [args], operation: 'not' });
    }
    rest = rest.replace(raw, '');
  });

  this.parseInstructions(rest).forEach((instr) => instrs.push(instr));

  return instrs;
}
```

We actually have 2 special instructions, one for `or` and another for `not` operations. Let’s check or example, but the same also applies to not operation.

Let’s imagine we need to find data where `firstName` is either John or Smith. Then we would use this RQL instruction `or[eq(firstName,John)eq(firstName,Smit)]`

#### Step 2: Extract limit and offset

We support actually 2 variants how to pass limit conditions

- `limit(100)` — we want the first 100 results, starting with offset 0 (therefore offset can be omitted).
- `limit(100, 10)` — we want 100 results, but skipping the first 10 results. In human language, we want results from 11th onwards.

To figure out what _limit_ and _offset_ should be used, let’s traverse over the instructions and find the one where operation is equal to limit . Let’s also validate that limit is not above max limit supported by the API, usually 100. It’s not a good idea to pull all data from your data source as this might bring unpredictable overload to your system (beside other challenges).

```typescript
private getLimitOffset(ctx: Context): Either<ContextAwareException, Context> {
  const ask = ctx.rqls.find((o) => o.operation === 'limit');
  if (!ask) {
    return Either.right(ctx);
  }

  const limit = ask.args[0];
  if (+limit > MAX_LIMIT) {
    return Either.left(
      InvalidValueException.create(
        'limit',
        `Limit must be a number lower or equal to ${MAX_LIMIT}`,
        400,
      ),
    );
  }

  return Either.right({ ...ctx, limit: +limit, offset: +(ask.args[1] || 0) });
}
```

#### Step 3: Extract information how to sort data
To be able to set how data should be sorted, you have to use sort operation. Data will be sorted by the specified column in ascending order. If you want to sort them in descending order, prefix the name of the column by — (dash)

- `sort(createdAt)` — data will be sorted by createdAt column in ascending order, meaning from 0 to 9
- `sort(-createdAt)` — data will be sorted by createdAt column in descending order, meaning from 9 to 0.


```typescript
private getOrderBy(ctx: Context): Either<ContextAwareException, Context> {
  const ask = ctx.rqls.find((o) => o.operation === 'sort');
  if (!ask) {
    return Either.right(ctx);
  }

  const orderBy = ask.args
    .filter((s) => s !== undefined)
    .map((s): Either<ContextAwareException, OrderBy> => {
      const direction = s.startsWith('-') ? 'desc' : 'asc';
      const key = s.startsWith('-') || s.startsWith('+') ? s.slice(1) : s;

      return !ctx.supportedColumns.includes(key)
        ? Either.left(
            InvalidValueException.create(
              'sort',
              `Invalid sort key: ${key}`,
              400,
            ),
          )
        : Either.right({ [key]: direction });
    });

  const leftValue = orderBy.find((d) => d.isLeft());
  if (leftValue) {
    return Either.left(leftValue.getLeft());
  }

  return Either.right({ ...ctx, orderBy: orderBy.map((d) => d.getRight()) });
}
```

#### Step 4: Let’s figure out the filters

It’s time to finally deal with search condition as we are done with limit and order part.

```typescript
private getFilters(ctx: Context): Either<ContextAwareException, Context> {
  const filters = ctx.rqls
    .filter((o) => o.operation !== 'sort')
    .filter((o) => o.operation !== 'limit');
  if (!filters.length) {
    return Either.right(ctx);
  }

  const where = filters.map((f) => this.mapFilter(f, ctx));

  const leftValue = where.find((d) => d.isLeft());
  return leftValue
    ? Either.left(leftValue.getLeft())
    : Either.right({ ...ctx, where: where.map((d) => d.getRight()) });
}
```

##### Step 4.1: Find filter instructions

The only thing we need to do is to iterate over the rql instructions and filter out those for sorting and for limit. We then remain with array of instructions, which provide conditions we use to find data.

##### Step 4.2: Map each instruction into a where statement

In first step, we find out all filter instruction and then map each one of them to a where statement. Let’s have a look how it’s done.

```typescript
private mapFilter(
  instr: RqlInstruction,
  ctx: Context,
): Either<ContextAwareException, RqlWhereOption> {
  return this.validateFilterValue(instr)
    .bind((instr) => this.validateOperation(instr))
    .bind((instr) => this.validateColumn(instr, ctx))
    .bind((instr) => this.toWhere(instr, ctx));
}
```

##### Step 4.2.1: Validate value of the instructions

As mentioned couple of times, we support different instructions and each of them have different conditions for their values. Let’s ensure we provided valid RQL statement.

```typescript
private validateFilterValue(
  instr: RqlInstruction,
): Either<ContextAwareException, RqlInstruction> {
  if (['and', 'or', 'not', 'null'].includes(instr.operation)) {
    return instr.args.length == 1
      ? Either.right(instr)
      : Either.left(
          MissingValueException.create(
            `filter:${instr.operation}`,
            'Filter has to have a value',
            400,
          ),
        );
  }

  if (instr.args.length != 2) {
    return Either.left(
      MissingValueException.create(
        `filter:${instr.operation}`,
        'Filter has to have a value',
        400,
      ),
    );
  }

  const operation = operationsMap[instr.operation as Operation];
  if (operation.type === 'string' && !isString(instr.args[1])) {
    return Either.left(
      InvalidValueException.create(
        `filter:${instr.operation}`,
        'Value has to be string',
        400,
      ),
    );
  }

  if (operation.type === 'number' && !isNumberString(instr.args[1])) {
    return Either.left(
      InvalidValueException.create(
        `filter:${instr.operation}`,
        'Value has to be numberic',
        400,
      ),
    );
  }

  return Either.right(instr);
}
```

##### Step 4.2.2: Validate the operation is supported

Let’s ensure that the operation is actually supported.

```typescript
const operationsMap: Record<Operation, { alias: string; type: string }> = {
  and: { alias: 'AND', type: 'string' },
  not: { alias: 'NOT', type: 'string' },
  contains: { alias: 'contains', type: 'string' },
  ends: { alias: 'endsWith', type: 'string' },
  eq: { alias: 'equals', type: 'string' },
  gt: { alias: 'gt', type: 'number' },
  gte: { alias: 'gte', type: 'number' },
  lt: { alias: 'lt', type: 'number' },
  lte: { alias: 'lte', type: 'number' },
  null: { alias: 'null', type: 'null' },
  or: { alias: 'OR', type: 'string' },
  starts: { alias: 'startsWith', type: 'string' },
};

private validateOperation(
  instr: RqlInstruction,
): Either<ContextAwareException, RqlInstruction> {
  return !operationsMap[instr.operation as Operation]
    ? Either.left(UnknownFilterException.create(`filter:${instr.operation}`))
    : Either.right(instr);
}
```

##### Step 4.2.3: Validate that column is supported

Let’s ensure that the column actually exists on the resource we are dealing with.

```typescript
private validateColumn(
  instr: RqlInstruction,
  ctx: Context,
): Either<ContextAwareException, RqlInstruction> {
  // We do not need to validate columns neither for AND, nor for OR and NOT operations
  // they are not dealing with the columns but rather with set of filters
  if (['and', 'or', 'not'].includes(instr.operation)) {
    return Either.right(instr);
  }

  const column = instr.args[0];

  return !ctx.supportedColumns.includes(column)
    ? Either.left(
        InvalidValueException.create(
          `filter:${instr.operation}:${column}`,
          `Property ${column} does not exist`,
          400,
        ),
      )
    : Either.right(instr);
}
```
##### Step 4.2.4: Map to where statement

Finally we are at position, where we can map the filter instruction into a where statement.

```typescript
private toWhere(
  instr: RqlInstruction,
  ctx: Context,
): Either<ContextAwareException, RqlWhereOption> {
  if (['and', 'or', 'not'].includes(instr.operation)) {
    const wheres = this.parseInstructions(instr.args[0]).map((f) =>
      this.mapFilter(f, ctx),
    );

    const leftValue = wheres.find((f) => f.isLeft());

    return leftValue
      ? Either.left(leftValue.getLeft())
      : Either.right({
          [instr.operation.toUpperCase()]: this.toWhereSingle(wheres),
        });
  }

  const operation = operationsMap[instr.operation as Operation];
  if (operation.type === 'number') {
    return Either.right({
      [instr.args[0]]: {
        [operationsMap[instr.operation as Operation].alias]: +instr.args[1],
      },
    });
  }

  if (operation.type === 'null') {
    return Either.right({
      [instr.args[0]]: null,
    });
  }

  return Either.right({
    [instr.args[0]]: {
      mode: 'insensitive',
      [operationsMap[instr.operation as Operation].alias]: instr.args[1],
    },
  });
}

private toWhereSingle(
  where: Either<ContextAwareException, RqlWhereOption>[],
) {
  return where
    .filter((wh) => wh.isRight())
    .map((wh) => {
      const rawWhere = wh.getRight();
      const key = Object.keys(rawWhere)[0];

    
      return rawWhere as RqlWhereType;
    });
}
```
#### Step 5: Map to the response

And we are finally done, we converted raw RQL into data we can use within our PRISMA repository.

```typescript
private toResponse(ctx: Context): Either<ContextAwareException, RqlResult> {
  return Either.right({
    hardLimit: ctx.hardLimit,
    limit: ctx.limit,
    offset: ctx.offset,
    orderBy: ctx.orderBy,
    where: ctx.where,
  });
}
```

Let’s see how it’s used from within Repository.

There are more things within the repository not tight to RQL, but I will not go over it.

```typescript
@Injectable()
export class OnboardingRepository extends AbstractRepository<Onboarding> {
  constructor(
    private readonly prisma: PrismaService,
    private readonly mapper: OnboardingMapper,
    private readonly rql: RqlService,
  ) {
    super();
  }

  async findAll(
    rql: string,
  ): Promise<Either<ContextAwareException, PaginatedResponse<Onboarding>>> {
    try {
      const columns: string[] = ObjectService.filteredProps(
        Prisma.OnboardingScalarFieldEnum,
        [],
      );

      return this.rql.parse(rql, columns).bindAsync(async (instrs) => {
        const where: Prisma.OnboardingWhereInput = {
          ...instrs.where.reduce((acc, curr) => ({ ...acc, ...curr }), {}),
        };

        const [data, totalRecords] = await Promise.all([
          this.prisma.onboarding.findMany({
            orderBy: instrs.orderBy,
            skip: instrs.offset,
            take: instrs.limit,
            where,
          }),
          this.prisma.onboarding.count({ where }),
        ]);

        return Either.right(
          new PaginatedResponse({
            totalsByStatus: StatusGroup.fromDbResults([], []),
            count: data.length,
            hardLimit: instrs.hardLimit,
            limit: instrs.limit,
            offset: instrs.offset,
            records: data,
            total: totalRecords,
          }),
        );
      });
    } catch (ex) {
      return Either.left(this.decorateException(ex));
    }
  }
}
```
