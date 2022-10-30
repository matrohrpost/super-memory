---
title: Complete Guide to NSwag and Autorest
date: 2022-10-31
---

# The complete guide: How to generate proper swagger.json for generating clients with autorest and c#

In our team we provide web APIs in order to access and data, that's nothing new. What we also provide are the generated clients for them as nuget packages, so other teams can grab the nuget package and do not have to generate the client for themselves.

We use [autorest]() for this purpose and well, autorest does not generate the code as we want it. In fact we have two pain points, we need to address:
1. Firstly, we want our model that all generated valuetype properties to be not nullable by default.
2. Second, we want the generated client to properly generate properly typed enums and the corresponding properties to it.

## Not nullable value types in generated client
The problem behind this is not the code generator of autorest. In fact we discovered that NSwag does not generate the ~required = true~ flag by default. Instead you have to NSwag to do so.

~~~csharp
// <summary>
/// Makes all value-type properties "Required" in the schema docs, which is appropriate since they cannot be null.
/// </summary>
/// <remarks>
/// This saves effort + maintenance from having to add <c>[Required]</c> to all value type properties; Web API, EF, and Json.net already understand
/// that value type properties cannot be null.
/// 
/// More background on the problem solved by this type: https://stackoverflow.com/questions/46576234/swashbuckle-make-non-nullable-properties-required </remarks>
public sealed class RequireValueTypePropertiesSchemaFilter : ISchemaFilter
{
    private readonly CamelCasePropertyNamesContractResolver _camelCaseContractResolver;

    /// <summary>
    /// Initializes a new <see cref="RequireValueTypePropertiesSchemaFilter"/>.
    /// </summary>
    /// <param name="camelCasePropertyNames">If <c>true</c>, property names are expected to be camel-cased in the JSON schema.</param>
    /// <remarks>
    /// I couldn't figure out a way to determine if the swagger generator is using <see cref="CamelCaseNamingStrategy"/> or not;
    /// so <paramref name="camelCasePropertyNames"/> needs to be passed in since it can't be determined.
    /// </remarks>
    public RequireValueTypePropertiesSchemaFilter(bool camelCasePropertyNames)
    {
        _camelCaseContractResolver = camelCasePropertyNames ? new CamelCasePropertyNamesContractResolver() : null;
    }

    /// <summary>
    /// Returns the JSON property name for <paramref name="property"/>.
    /// </summary>
    /// <param name="property"></param>
    /// <returns></returns>
    private string PropertyName(PropertyInfo property)
    {
        return _camelCaseContractResolver?.GetResolvedPropertyName(property.Name) ?? property.Name;
    }

    /// <summary>
    /// Adds non-nullable value type properties in a <see cref="Type"/> to the set of required properties for that type.
    /// </summary>
    /// <param name="model"></param>
    /// <param name="context"></param>
    public void Apply(OpenApiSchema model, SchemaFilterContext context)
    {
        foreach (var property in context.Type.GetProperties())
        {
            var schemaPropertyName = PropertyName(property);
            // This check ensures that properties that are not in the schema are not added as required.
            // This includes properties marked with [IgnoreDataMember] or [JsonIgnore] (should not be present in schema or required).
            if (model.Properties?.ContainsKey(schemaPropertyName) == true)
            {
                // Value type properties are required,
                // except: Properties of type Nullable<T> are not required.
                var propertyType = property.PropertyType;
                if (propertyType.IsValueType
                    && ! (propertyType.IsConstructedGenericType && (propertyType.GetGenericTypeDefinition() == typeof(Nullable<>))))
                {
                    // Properties marked with [Required] are already required (don't require it again).
                    if (! property.CustomAttributes.Any(attr =>
                                                        {
                                                            var t = attr.AttributeType;
                                                            return t == typeof(RequiredAttribute);
                                                        }))
                    {
                        // Make the value type property required
                        if (model.Required == null)
                        {
                            model.Required = new HashSet<string>();
                        }
                        model.Required.Add(schemaPropertyName);
                    }
                }
            }
        }
    }
}
~~~

Additionally you have to add the filter to the swagger generation.

~~~csharp
public void ConfigureServices(IServiceCollection services) 
{
    services.AddSwaggerGen(c =>
    {
      c.SchemaFilter<RequireValueTypePropertiesSchemaFilter>();
    }
}
~~~


