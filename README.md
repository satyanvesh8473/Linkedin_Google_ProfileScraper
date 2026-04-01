using System;
using System.Collections.Generic;
using System.Text.Json;
using System.Text.Json.Serialization;
using MongoDB.Bson;
using MongoDB.Bson.Serialization;
using MongoDB.Bson.Serialization.Attributes;
using MongoDB.Bson.Serialization.Serializers;

namespace RulesUpload.CreateRules
{
    [BsonIgnoreExtraElements]
    public class CreateRulesRequest
    {
        [BsonElement("model_id")]
        [JsonPropertyName("model_id")]
        public string ModelId { get; set; }

        [BsonElement("hash_code")]
        [JsonPropertyName("hash_code")]
        public string HashCode { get; set; }

        [BsonElement("rules")]
        [JsonPropertyName("rules")]
        public List<CreateRuleParameters> Rules { get; set; }
    }

    [BsonIgnoreExtraElements]
    public class CreateRuleParameters
    {
        [BsonElement("id")]
        [JsonPropertyName("id")]
        public string Id { get; set; }

        [BsonElement("state")]
        [JsonPropertyName("state")]
        public string State { get; set; }

        [BsonElement("country")]
        [JsonPropertyName("country")]
        public string Country { get; set; }

        [BsonElement("type")]
        [JsonPropertyName("type")]
        public string RuleType { get; set; }

        [BsonElement("reportable")]
        [JsonPropertyName("reportable")]
        public bool Reportable { get; set; }

        [BsonElement("score")]
        [JsonPropertyName("score")]
        public string Score { get; set; }

        [BsonElement("points")]
        [JsonPropertyName("points")]
        public double Points { get; set; }

        [BsonElement("frequency")]
        [JsonPropertyName("frequency")]
        public int Frequency { get; set; }

        [BsonElement("label")]
        [JsonPropertyName("label")]
        public string Label { get; set; }

        [BsonElement("label_rank")]
        [JsonPropertyName("label_rank")]
        public int LabelRank { get; set; }

        [BsonElement("expressions")]
        [JsonPropertyName("expressions")]
        public RuleExpressions Expressions { get; set; }

        [BsonElement("complexExpressions")]
        [JsonPropertyName("complexExpressions")]
        public string ExpressionString { get; set; }


        public RuleModel ToRuleModel()
        {
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
              //Expressions = Expressions,
                ExpressionString = ExpressionString,
                Expressions = MongoDB.Bson.Serialization.BsonSerializer.Deserialize<BsonDocument>(JsonSerializer.Serialize(Expressions))
            };
        }
    }   

    [BsonIgnoreExtraElements]
    public class RuleExpressions : ICloneable
    {
        [BsonElement("operator")]
        [JsonPropertyName("operator")]
        public string Operator { get; set; }

        [JsonPropertyName("left")]
        [BsonElement]
        public object Left { get; set; }

        [JsonPropertyName("right")]
        [BsonElement]
        public object Right { get; set; }

        //[BsonIgnore]
        //[BsonElement]
        //public RuleExpressions LeftRuleExpressions { get; set; }

       // [BsonIgnore]
       // [BsonElement]
        //public RuleExpressions RightRuleExpressions { get; set; }

        //[BsonElement("left")]
        //[JsonIgnore]
        // [BsonSerializer(typeof(MyCustomStringSerializer))]
        // public object LeftBson { get; set; }        
       // public BsonDocument LeftBson { get; set; }

        //[BsonElement("right")]
        //[JsonIgnore]
         //[BsonSerializer(typeof(MyCustomStringSerializer))]
         /// <summary>
         /// public object RightBson { get; set; }       
         /// </summary>
         /// <returns></returns>
        //public BsonDocument RightBson { get; set; }

        public object Clone()
        {
            return this.MemberwiseClone();
        }
    }
    class MyCustomStringSerializer : IBsonSerializer
    {
        public object Deserialize(BsonDeserializationContext context, BsonDeserializationArgs args)
        {            
            return BsonValueSerializer.Instance.Deserialize(context);
        }
        public void Serialize(BsonSerializationContext context, BsonSerializationArgs args, object value)
        {
            switch (value)
            {
                case null:
                    context.Writer.WriteNull();
                    break;
                case BsonDocument doc:
                    {
                        context.Writer.WriteStartDocument();
                        var elementCount = doc.ElementCount;
                        for (var index = 0; index < elementCount; index++)
                        {
                            var element = doc.GetElement(index);
                            context.Writer.WriteName(element.Name);
                            BsonValueSerializer.Instance.Serialize(context, element.Value);
                        }
                        context.Writer.WriteEndDocument();
                        break;
                    }
                case string str:
                    {
                        context.Writer.WriteString(str);
                        break;
                    }                    
                // TODO: check other types
                default:
                    BsonSerializer.Serialize(context.Writer, value.ToString());
                    break;
            }
        }
        public Type ValueType => typeof(object);
    }
}
