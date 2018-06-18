# Habilitando paginação (condicional) e query options

### Uso:

Ex.: {link}?pageSize=20
```json
{
    "@odata.context":"{link}/$metadata#...",
    "value":[...],
    "@odata.nextLink":"{link}?pageSize=20&$skip=20"
}
```

Ex.: {link}?pageSize=20&$count=true
```json
{
    "@odata.context":"{link}/$metadata#...",
    "@odata.count":45,
    "value":[...],
    "@odata.nextLink":"{link}?pageSize=20&$count=true&$skip=20"
```

Ex.: {link}

```json
{
    "@odata.context":"{link}/$metadata#...",
    "value":[
        ...
    ]
}
```

## Implementação - Usando atributo

```csharp
namespace Boiko
{
    public class EnablePageQueryAttribute : EnableQueryAttribute
    {
        public string PageSizeQueryName { get; set; } = "pagesize";

        public override void OnActionExecuted(HttpActionExecutedContext actionExecutedContext)
        {
            if (TryGetPageSize(actionExecutedContext.Request, out var pageSize))
            {
                var queryAttribute = new EnableQueryAttribute
                {
                    PageSize = pageSize,
                    AllowedQueryOptions = AllowedQueryOptions,
                    MaxTop = MaxTop,
                    AllowedArithmeticOperators = AllowedArithmeticOperators,
                    AllowedFunctions = AllowedFunctions,
                    AllowedLogicalOperators = AllowedLogicalOperators,
                    AllowedOrderByProperties = AllowedOrderByProperties,
                    EnableConstantParameterization = EnableConstantParameterization,
                    EnsureStableOrdering = EnsureStableOrdering,
                    HandleNullPropagation = HandleNullPropagation,
                    MaxAnyAllExpressionDepth = MaxAnyAllExpressionDepth,
                    MaxExpansionDepth = MaxExpansionDepth,
                    MaxNodeCount = MaxNodeCount,
                    MaxOrderByNodeCount = MaxOrderByNodeCount,
                    MaxSkip = MaxSkip,
                };

                queryAttribute.OnActionExecuted(actionExecutedContext);
            }
            else
            {
                base.OnActionExecuted(actionExecutedContext);
            }
        }

        private bool TryGetPageSize(HttpRequestMessage request, out int pageSize)
        {
            pageSize = default(int);

            if (PageSize > 0)
            {
                return false;
            }

            var queryString = request.GetQueryNameValuePairs();
            var pageSizeQueries = queryString
                .Where(q => q.Key.Equals(PageSizeQueryName, StringComparison.OrdinalIgnoreCase));

            foreach (var pageSizeQuery in pageSizeQueries)
            {
                if (int.TryParse(pageSizeQuery.Value, out pageSize))
                {
                    return true;
                }
            }

            return false;
        }
    }
}
```

#### Usando o atributo

```csharp
namespace Boiko
{
    public class ExempleController : ODataController
    {
        [EnablePageQuery]
        public IHttpActionResult Get() {
            return Service.GetAll();
        }
    }
}
```

## Implementação - Usando código

```csharp
using System.Web.OData.Query;

namespace Boiko
{
    public class ExampleController : ODataController
    {
        public IHttpActionResult Get(ODataQueryOptions<T> queryOptions, int? pageSize = null)
        {
            var queryResult = Service.GetAll();

            if (pageSize == null)
            {
                return Ok(queryOptions.ApplyTo(queryResult));
            }

            var count = queryResult.Count();
            var settings = new ODataQuerySettings
            {
                PageSize = pageSize.Value,
            };

            queryResult = queryOptions.ApplyTo(queryResult, settings);
            var pageResult = new PageResult<T>(queryResult as IEnumerable<T>, Request.GetNextPageLink(pageSize.Value), count);

            return Ok(pageResult);
        }
    }
}
```
