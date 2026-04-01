private void TraverseJsonElement(RuleExpressions root, CreateRuleParameters createRuleParameters, CreateRulesRequest createRulesRequest)
        {
            if (root.Operator != null)
            {
                NullCheck(root);
                var leftElement = (JsonElement)root.Left;
                var rightElement = (JsonElement)root.Right;
                RuleExpressions leftExpr = new RuleExpressions(), rightExpr = new RuleExpressions();
                if (leftElement.ValueKind != JsonValueKind.String)
                {
                    string leftvalue = string.Empty, rightvalue;
                    leftExpr = JsonSerializer.Deserialize<RuleExpressions>(leftElement.GetRawText(), new JsonSerializerOptions
                    {
                        MaxDepth = 256
                    });
                    NullCheck(leftExpr);
                    var LeftleftElement = (JsonElement)leftExpr.Left;
                    if (LeftleftElement.ValueKind == JsonValueKind.String)
                    {
                        leftvalue = leftExpr.Left.ToString();
                        rightvalue = leftExpr.Right.ToString();
                        
                    }
                }
                if (rightElement.ValueKind != JsonValueKind.String && rightElement.ValueKind != JsonValueKind.Number)
                {
                    string leftvalue = string.Empty;
                    rightExpr = JsonSerializer.Deserialize<RuleExpressions>(rightElement.GetRawText(), new JsonSerializerOptions
                    {
                        MaxDepth = 256
                    });
                    NullCheck(rightExpr);

                    var rightleftElement = (JsonElement)rightExpr.Left;
                    if (rightleftElement.ValueKind == JsonValueKind.String)
                    {
                        leftvalue = rightExpr.Left.ToString();

                       
                    }
                    TraverseJsonElement(leftExpr, createRuleParameters, createRulesRequest);
                    TraverseJsonElement(rightExpr, createRuleParameters, createRulesRequest);
                }

            }
        }
