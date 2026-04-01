See the end of this message for details on invoking 
just-in-time (JIT) debugging instead of this dialog box.
 
************** Exception Text **************
System.InvalidOperationException: CurrentDepth (64) is equal to or larger than the maximum allowed depth of 64. Cannot write the next JSON object or array.
   at System.Text.Json.ThrowHelper.ThrowInvalidOperationException(ExceptionResource resource, Int32 currentDepth, Int32 maxDepth, Byte token, JsonTokenType tokenType)
   at System.Text.Json.Utf8JsonWriter.WriteStart(Byte token)
   at System.Text.Json.JsonDocument.WriteComplexElement(Int32 index, Utf8JsonWriter writer)
   at System.Text.Json.JsonDocument.WriteElementTo(Int32 index, Utf8JsonWriter writer)
   at System.Text.Json.Serialization.Converters.JsonElementConverter.Write(Utf8JsonWriter writer, JsonElement value, JsonSerializerOptions options)
   at System.Text.Json.Serialization.JsonConverter`1.TryWrite(Utf8JsonWriter writer, T& value, JsonSerializerOptions options, WriteStack& state)
   at System.Text.Json.Serialization.JsonConverter`1.TryWriteAsObject(Utf8JsonWriter writer, Object value, JsonSerializerOptions options, WriteStack& state)
   at System.Text.Json.Serialization.JsonConverter`1.TryWrite(Utf8JsonWriter writer, T& value, JsonSerializerOptions options, WriteStack& state)
   at System.Text.Json.Serialization.Metadata.JsonPropertyInfo`1.GetMemberAndWriteJson(Object obj, WriteStack& state, Utf8JsonWriter writer)
   at System.Text.Json.Serialization.Converters.ObjectDefaultConverter`1.OnTryWrite(Utf8JsonWriter writer, T value, JsonSerializerOptions options, WriteStack& state)
   at System.Text.Json.Serialization.JsonConverter`1.TryWrite(Utf8JsonWriter writer, T& value, JsonSerializerOptions options, WriteStack& state)
   at System.Text.Json.Serialization.Metadata.JsonPropertyInfo`1.GetMemberAndWriteJson(Object obj, WriteStack& state, Utf8JsonWriter writer)
   at System.Text.Json.Serialization.Converters.ObjectDefaultConverter`1.OnTryWrite(Utf8JsonWriter writer, T value, JsonSerializerOptions options, WriteStack& state)
   at System.Text.Json.Serialization.JsonConverter`1.TryWrite(Utf8JsonWriter writer, T& value, JsonSerializerOptions options, WriteStack& state)
   at System.Text.Json.Serialization.Converters.ListOfTConverter`2.OnWriteResume(Utf8JsonWriter writer, TCollection value, JsonSerializerOptions options, WriteStack& state)
   at System.Text.Json.Serialization.JsonCollectionConverter`2.OnTryWrite(Utf8JsonWriter writer, TCollection value, JsonSerializerOptions options, WriteStack& state)
