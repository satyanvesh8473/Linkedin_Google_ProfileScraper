 public static bool IsValid(CreateRuleParameters rule)
        {
            if (rule.Expressions.Operator == null || rule.RuleType == null)
            {
                throw new ApplicationException("Rule Type Parameter value");
            }
            if (!IsValidRuleType(rule.RuleType))
            {
                throw new ApplicationException(string.Format(" Rule Type {0}", rule.RuleType));
            }
            NullValueCheck(rule.Expressions);
            var leftElement = (JsonElement)rule.Expressions.Left;
            var rightElement = (JsonElement)rule.Expressions.Right;
            if (leftElement.ValueKind == JsonValueKind.String || leftElement.ValueKind == JsonValueKind.Number)
            {
                if (rule.Expressions.Operator == null)
                {
                    throw new ApplicationException("Relational Operator Value null");
                }
                string leftvalue = leftElement.ToString();
                if (!Enum.IsDefined(typeof(RelationalOperator), rule.Expressions.Operator))
                {
                    throw new ApplicationException(string.Format(" Relational operator {0}", rule.Expressions.Operator.ToString()));
                }
                string rightvalue = rightElement.ToString();
                LeftRightCheck(leftvalue, rightvalue);
                if ((leftvalue.EndsWith("Age") || leftvalue == "ConfinementDuration" || leftvalue == "Salary") && (leftElement.ValueKind != JsonValueKind.Number))
                {
                    throw new ApplicationException(string.Format("Right Value is Invalid for Left Value {0}", leftvalue.ToString()));
                }
            }
            else
            {
                Preoder(rule.Expressions);
            }
            return true;
        }
