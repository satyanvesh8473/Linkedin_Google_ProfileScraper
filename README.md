private BsonValue ConvertToBson(object value)
{
    if (value == null)
        return BsonNull.Value;

    if (value is RuleExpressions expr)
    {
        return new BsonDocument
        {
            { "operator", expr.Operator ?? "" },
            { "left", ConvertToBson(expr.Left) },
            { "right", ConvertToBson(expr.Right) }
        };
    }

    if (value is string s)
        return new BsonString(s);

    if (value is int i)
        return new BsonInt32(i);

    if (value is double d)
        return new BsonDouble(d);

    if (value is bool b)
        return new BsonBoolean(b);

    return new BsonString(value.ToString());
}
