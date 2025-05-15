+++
title = 'Nestjs Swagger Made Easy'
date = '2025-03-11T07:50:41+01:00'
tags = ['nestjs', 'swagger']
draft = true
+++

[NestJS](https://nestjs.com/) is a great tool to run your API, [Swagger](https://swagger.io/docs/) is a great tool to document it, but adding this two together might bring a lot of duplications.

In the following page, we will come up with solution, that we remove the duplication without losing any power that comes with Swagger.

## Background

I assume, that you already have swagger and NestJS glued together. There are plenty of instructions out there already.

## Standart way

When you want to use Swagger to document your endpoint, you will usually go to your controller and add piece of code that looks like this.

```typescript
@ApiOperation({
    summary: 'Searches among applicant audits',
    description: 'The enpoint takes a query defined in the parameter and uses it as base to find data within your applicant audits',
    responses: {
        '200': {
            description: 'Success response',
            content: {
                'application/json': {
                    schema: {
                        $ref: `#/components/schemas/ApplicantAuditResponse`,
                    },
                },
            }
        },
        '4xx': {
            description: 'Error response',
            content: {
                'application/json': {
                    schema: {
                        $ref: `#/components/schemas/ErrorResponse`,
                    },
                },
            }
        },
        '5xx': {
            description: 'Error response',
            content: {
                'application/json': {
                    schema: {
                        $ref: `#/components/schemas/ErrorResponse`,
                    },
                },
            }
        },
    },
})
@Get('/applicants')
applicantsAudit(@Query('q') rql: string) {
    return this.applicantQuery.execute(rql);
}
```

And let's imagine, that we have another function within the same controller. We will most definately copy most of the code with very small modifications. We can do better than this, right?

## Power of helpers

If we really think about this, we usually need to change these things

- summary of the endpoint
- des of the enpoint
- reference of the success endpoint

The rest of the data stays the same, so we can remove the duplications

### SwaggerDocs helper

Let's write a short class called SwaggerDocs helper and let's it take care about the repetetive work.

```typescript
export class SwaggerDocs {
  static success(
    ref: string,
    code: number = 200,
  ): Record<string, ResponseObject> {
    return {
      [code]: {
        description: 'Successful response',
        content: this.content(ref),
      },
    };
  }

  static errors() {
    return {
      ...this.error('4xx'),
      ...this.error('5xx'),
    };
  }

  static error(code: string, ref: string = 'ErrorResponse') {
    return {
      [code]: {
        description: 'Error response',
        content: this.content(ref),
      },
    };
  }

  private static content(ref: string): ContentObject {
    return {
      'application/json': {
        schema: {
          $ref: `#/components/schemas/${ref}`,
        },
      },
    };
  }
}
```

Having SwaggerDocs in place, we can change the code to following

```typescript
@ApiOperation({
    summary: 'Searches among applicant audits',
    responses: {
      ...SwaggerDocs.success('ApplicantAuditResponse'),
      ...SwaggerDocs.errors(),
    },
})
@Get('/applicants')
applicantsAudit(@Query('q') rql: string) {
return this.applicantQuery.execute(rql);
}
```

And we are done. The code is much more concise, easier to mantained and way less code is needed to be written. 

