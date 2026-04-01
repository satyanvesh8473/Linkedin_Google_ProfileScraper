public RuleModel ToRuleModel()
{
    var options = new JsonSerializerOptions
    {
        ReferenceHandler = ReferenceHandler.IgnoreCycles,
        MaxDepth = 256
    };

    return new RuleModel
    {
        Country = Country,
        Frequency = Frequency,
        Id = Id,
        Label = Label,
        LabelRank = LabelRank,
        Points = Points,
        Reportable = Reportable,
        RuleType = RuleType,
        Score = Score,
        State = State,
        ExpressionString = ExpressionString,
        Expressions = MongoDB.Bson.Serialization.BsonSerializer
            .Deserialize<BsonDocument>(JsonSerializer.Serialize(Expressions, options))
    };
}
