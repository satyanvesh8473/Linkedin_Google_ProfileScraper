using MongoDB.Bson;
using MongoDB.Bson.Serialization.Attributes;
using MongoDB.Driver;
using System;
using System.Collections.Generic;
using System.Globalization;
using System.IO;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using System.Text.Json.Serialization;
using System.Threading.Tasks;

namespace RulesUpload.CreateRules
{
    class CreateRuleManager
    {        
        public Task Validate(string path, int env, EventHandler<StatusChangeArgs> onChangeHandler)
        {
            string errorFilePath = path.Replace(".csv", "") + "_Error.csv";
            if (File.Exists(errorFilePath))
            {
                File.Delete(errorFilePath);
            }
            var errFile = File.Create(errorFilePath);
            errFile.Close();
            
            onChangeHandler(this, new StatusChangeArgs(false, "Parsing file..."));

            CreateRulesRequest requestObject = new CreateRulesRequest()
            {
                Rules = new List<CreateRuleParameters>()
            };

            var lineList = File.ReadAllLines(path);            
            string errLine = string.Empty;

            for (int i = 0; i < lineList.Length; i++)
            {
                
                if (i == 0)
                {
                    // header columns ignore
                    continue;
                }

                List<string> columnList = lineList[i].Split(',').ToList();               

                if (columnList.Count != 11)
                {
                    errLine = "Invalid Row " + (i + 1) + " expect to have 11 coulumns but has " + columnList.Count  + Environment.NewLine;
                    File.AppendAllText(errorFilePath, errLine);
                    continue;
                }

               // ValidateRulePropertyManager.Load(env);
               // ChargeLevelMappingManager.Load(env);

                string ruleType = columnList[0];
                bool reportable = bool.Parse(columnList[1]);
                string score = columnList[2];
                string label = columnList[3];
                int labelRank = !string.IsNullOrWhiteSpace(columnList[4]) ? int.Parse(columnList[4]) : 0;
                int frequency = !string.IsNullOrWhiteSpace(columnList[5]) ? int.Parse(columnList[5]) : 0;
                double points = !string.IsNullOrWhiteSpace(columnList[6]) ? double.Parse(columnList[6]) : 0;
                string state = columnList[7];
                string country = columnList[8];
                RuleExpressions expressions = BuildExpression_Validate(columnList[9], "&&", errorFilePath, i);
                try
                {
                    requestObject.Rules.Add(
                        new CreateRuleParameters()
                        {
                            RuleType = ruleType,
                            Reportable = reportable,
                            Score = score,
                            Label = label,
                            LabelRank = labelRank,
                            Frequency = frequency,
                            Points = points,
                            State = state,
                            Country = country,
                            Expressions = expressions
                        });
                }
                catch (System.Exception ex)
                {
                    onChangeHandler(this, new StatusChangeArgs(true, ex.ToString()));

                    errLine = "Invalid Row " + (i + 1) + ex.ToString() + Environment.NewLine;
                    File.AppendAllText(errorFilePath, errLine);
                    continue;                   
                }
            }

            onChangeHandler(this, new StatusChangeArgs(false, "Compiling rule set..."));
            var ruleTypeSet = new HashSet<string>();
            int j = 0;
            foreach (var rule in requestObject.Rules)
            {
                j = j + 1;
                ruleTypeSet.Add(rule.RuleType);
                if (ruleTypeSet.Count > 1)
                {                    
                    File.AppendAllText(errorFilePath, "RuleType should be unique.");
                    break;
                }
               // CompileExpressions(rule.Expressions);
              //  RulePayLoadValidationForErrorFile.IsValid_Validate(rule, errorFilePath, j);
            }

            var csvFileLenth = new System.IO.FileInfo(errorFilePath).Length;
            if (csvFileLenth > 0)
            {
                onChangeHandler(this, new StatusChangeArgs(false, "Completed Validation, please check error file..."));                
            }
            else
            {
                onChangeHandler(this, new StatusChangeArgs(false, "Completed Validation..."));
                if (File.Exists(errorFilePath))
                {                    
                    File.Delete(errorFilePath);
                }
            }
            return Task.CompletedTask;
        }
        public Task Create(string path, int env, EventHandler<StatusChangeArgs> onChangeHandler)
        {
            onChangeHandler(this, new StatusChangeArgs(false, "Parsing file..."));

            CreateRulesRequest requestObject = new CreateRulesRequest()
            {
                Rules = new List<CreateRuleParameters>()
            };

            var lineList = File.ReadAllLines(path);

            for (int i = 0; i < lineList.Length; i++)
            {
                if (i == 0)
                {
                    // header columns ignore
                    continue;
                }

                List<string> columnList = lineList[i].Split(',').ToList();

                if (columnList.Count != 11)
                {
                    onChangeHandler(this, new StatusChangeArgs(true, "Invalid Row " + (i + 1) +
                                                                     " expect to have 11 columns but has " + columnList.Count));
                    return Task.CompletedTask;
                }

                string ruleType = columnList[0];
                bool reportable = bool.Parse(columnList[1]);
                string score = columnList[2];
                string label = columnList[3];
                int labelRank = !string.IsNullOrWhiteSpace(columnList[4]) ? int.Parse(columnList[4]) : 0;
                int frequency = !string.IsNullOrWhiteSpace(columnList[5]) ? int.Parse(columnList[5]) : 0;
                double points = !string.IsNullOrWhiteSpace(columnList[6]) ? double.Parse(columnList[6]) : 0;
                string state = columnList[7];
                string country = columnList[8];
                RuleExpressions expressions = BuildExpression(columnList[9], "&&");
                if (expressions == null)
                {
                    onChangeHandler(this, new StatusChangeArgs(true, "Could not create a rule as Expression is Null or Blank"));
                    //MessageBox.Show("Expression is Null or Blank");

                    return Task.CompletedTask;
                }
                try
                {
                    requestObject.Rules.Add(
                        new CreateRuleParameters()
                        {
                            RuleType = ruleType,
                            Reportable = reportable,
                            Score = score,
                            Label = label,
                            LabelRank = labelRank,
                            Frequency = frequency,
                            Points = points,
                            State = state,
                            Country = country,
                            Expressions = expressions,
                            ExpressionString = columnList[9]
                        });
                }
                catch (System.Exception ex)
                {
                    onChangeHandler(this, new StatusChangeArgs(true, ex.ToString()));
                    return Task.CompletedTask;
                }
            }

            onChangeHandler(this, new StatusChangeArgs(false, "Loading charge level mapping..."));

            ChargeLevelMappingManager.Load(env);

            var options = new JsonSerializerOptions
            {
                MaxDepth = 512
            };
            string requestData = JsonSerializer.Serialize<CreateRulesRequest>(requestObject, options);

            CreateRulesRequest requestObjectB = JsonSerializer.Deserialize<CreateRulesRequest>(requestData,
                new JsonSerializerOptions
                {
                    MaxDepth = 512
                });
            var modelId = CreateRule(requestObjectB, out bool isModelIdPresent, onChangeHandler);

            if (isModelIdPresent)
            {
                var args = new StatusChangeArgs(true, "");
                args.Exists = true;
                args.ModelId = modelId;
                onChangeHandler(this, args);
            }
            else
            {
                var args = new StatusChangeArgs(true, "");
                args.Exists = false;
                args.ModelId = modelId;
                onChangeHandler(this, args);
            }

            return Task.CompletedTask;
        }

        ChargeSeverityValidation objChargeLevelMapping = new ChargeSeverityValidation();

        private string CreateRule(CreateRulesRequest obj, out bool isModelIdPresent, EventHandler<StatusChangeArgs> onChangeHandler)
        {
            Boolean isSimpleExpression = true;
            onChangeHandler(this, new StatusChangeArgs(false, "Validating rule set..."));
            objChargeLevelMapping.ListOfChargeSeverity(obj, isSimpleExpression);
            return SaveRule(obj, out isModelIdPresent);
        }

        /*private static void CompileExpressions(RuleExpressions rule)
        {
            if (rule.LeftRuleExpressions == null)
            {
                var xLeftElement = (JsonElement)rule.Left;
                if (xLeftElement.ValueKind == JsonValueKind.Object)
                {
                    var leftJson = xLeftElement.GetRawText();
                    var xLeftExpr = JsonSerializer.Deserialize<RuleExpressions>(leftJson,
                        new JsonSerializerOptions
                        {
                            MaxDepth = 512
                        });
                    rule.LeftRuleExpressions = xLeftExpr;
                    rule.LeftBson = BsonDocument.Parse(leftJson);
                    CompileExpressions(rule.LeftRuleExpressions);
                }
                else if (xLeftElement.ValueKind == JsonValueKind.String)
                {
                    rule.LeftBson = xLeftElement.GetString();
                }
                else if (xLeftElement.ValueKind == JsonValueKind.Number)
                {
                    rule.LeftBson = xLeftElement.GetDouble();
                }
            }

            if (rule.RightRuleExpressions == null)
            {
                var xRightElement = (JsonElement)rule.Right;
                if (xRightElement.ValueKind == JsonValueKind.Object)
                {
                    var rightJson = xRightElement.GetRawText();
                    var xRightExpr = JsonSerializer.Deserialize<RuleExpressions>(rightJson,
                        new JsonSerializerOptions
                        {
                            MaxDepth = 512
                        });
                    rule.RightRuleExpressions = xRightExpr;
                    rule.RightBson = BsonDocument.Parse(rightJson);
                    CompileExpressions(rule.RightRuleExpressions);
                }
                else if (xRightElement.ValueKind == JsonValueKind.String)
                {
                    rule.RightBson = xRightElement.GetString();
                }
                else if (xRightElement.ValueKind == JsonValueKind.Number)
                {
                    rule.RightBson = xRightElement.GetDouble();
                }
            }
        }*/

        private string SaveRule(CreateRulesRequest obj, out bool isModelIdPresent)
        {
            isModelIdPresent = true;
            var ruleTypeSet = new HashSet<string>();
            foreach (var rule in obj.Rules)
            {
                ruleTypeSet.Add(rule.RuleType);
                if (ruleTypeSet.Count > 1)
                {
                    throw new ApplicationException("DuplicateRuleTypeException");
                }
                //CompileExpressions(rule.Expressions);

                RulePayLoadValidation.IsValid(rule);
            }
            obj.Rules.Sort(new RuleComparer());
            string hashCode = HashCodeGenerate.ToSha512(obj.Rules);
            var modelId = ChargeLevelMappingManager.GetExistingModelId(hashCode);
            if (!string.IsNullOrEmpty(modelId)) return modelId;

            isModelIdPresent = false;
            modelId = $"{obj.Rules[0].RuleType}-{Guid.NewGuid()}";

            var rules = new List<RuleModel>(obj.Rules.Count);

            foreach (var rule in obj.Rules)
                {
                    rule.Id = Guid.NewGuid().ToString();
                    var r = rule.ToRuleModel();
                    r.ModelId = modelId;
                    r.HashCode = hashCode;
                    rules.Add(r);
                }

                ChargeLevelMappingManager.Save(rules);
        
            return modelId;
        }
        /*private string SaveRule(CreateRulesRequest obj, out bool isModelIdPresent, EventHandler<StatusChangeArgs> onChangeHandler)
        {
            isModelIdPresent = true;
            onChangeHandler(this, new StatusChangeArgs(false, "Compiling rule set..."));
            var ruleTypeSet = new HashSet<string>();
            foreach (var rule in obj.Rules)
            {
                ruleTypeSet.Add(rule.RuleType);
                if (ruleTypeSet.Count > 1)
                {
                    throw new ApplicationException("DuplicateRuleTypeException");
                }
                CompileExpressions(rule.Expressions);

                RulePayLoadValidation.IsValid(rule);
            }
            obj.Rules.Sort(new RuleComparer());
            string hashCode = HashCodeGenerate.ToSha512(obj.Rules);
            onChangeHandler(this, new StatusChangeArgs(false, "Checking if rule set already exist..."));
            var modelId = ChargeLevelMappingManager.GetExistingModelId(hashCode);
            if (!string.IsNullOrEmpty(modelId)) return modelId;

            isModelIdPresent = false;
            modelId = $"{obj.Rules[0].RuleType}-{Guid.NewGuid()}";

            var rules = new List<RuleModel>(obj.Rules.Count);

            foreach (var rule in obj.Rules)
            {
                rule.Id = Guid.NewGuid().ToString();
                var r = rule.ToRuleModel();
                r.ModelId = modelId;
                r.HashCode = hashCode;
                rules.Add(r);
            }

            onChangeHandler(this, new StatusChangeArgs(false, "Saving rules..."));

            ChargeLevelMappingManager.Save(rules);

            return modelId;
        }*/
        public RuleExpressions BuildExpression(string expression, string symbol)
        {
            // List<string> symbols = new List<string> { "&&", "||"};
            if (string.IsNullOrEmpty(expression))
            {
                return null;
            }

            var expressionList = expression.Split(new[] {symbol}, 2, System.StringSplitOptions.RemoveEmptyEntries)
                .ToList();

            var expressionList1 = expression.Split(new string[] { "&&", "||" }, 2, System.StringSplitOptions.RemoveEmptyEntries)
               .ToList();

            if (expressionList.Count == 0)
            {
                return new RuleExpressions();
            }

            if (expressionList.Count == 1)
            {
                if (expressionList[0].Contains("&&"))
                {
                    return BuildExpression(expressionList[0], "&&");
                }

                if (expressionList[0].Contains("||"))
                {
                    return BuildExpression(expressionList[0], "||");
                }

                return BuildExpression_Operator(expressionList[0]);
            }

            return new RuleExpressions()
            {
                Left = BuildExpression(expressionList[0], "&&"),
                Operator = symbol == "&&" ? "And" : "Or",
                Right = BuildExpression(expressionList[1], "&&")
            };
        }
        public RuleExpressions BuildExpression_Validate(string expression, string symbol, string errFilePath, int rowNumber)
        {
            var expressionList = expression.Split(new[] { symbol }, 2, System.StringSplitOptions.RemoveEmptyEntries)
                .ToList();

            var objRuleExpressions = new RuleExpressions();

            if (expressionList.Count == 0)
            {
                return objRuleExpressions;
            }

            if (expressionList.Count == 1)
            {
                if (expressionList[0].Contains("&&"))
                {
                    return BuildExpression_Validate(expressionList[0], "&&", errFilePath, rowNumber);
                }

                if (expressionList[0].Contains("||"))
                {
                    return BuildExpression_Validate(expressionList[0], "||", errFilePath, rowNumber);
                }

                return BuildExpression_Operator_Validate(expressionList[0], errFilePath, rowNumber);                
            }

            return new RuleExpressions()
            {
                Left = BuildExpression_Validate(expressionList[0], "&&", errFilePath, rowNumber),
                Operator = symbol == "&&" ? "And" : "Or",
                Right = BuildExpression_Validate(expressionList[1], "&&", errFilePath, rowNumber)
            };
        }

        private RuleExpressions BuildExpression_Operator_Validate(string expression, string errFilePath, int rowNumber)
        {
            List<string> expressionList_CompareA = System.Text.RegularExpressions.Regex.Split(expression, "greaterthanorequal|>=", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareB = System.Text.RegularExpressions.Regex.Split(expression, "lessthanorequal|<=", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareC = System.Text.RegularExpressions.Regex.Split(expression, "notequal|!=", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareD = System.Text.RegularExpressions.Regex.Split(expression, "equal|=", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareE = System.Text.RegularExpressions.Regex.Split(expression, "greaterthan|>", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareF = System.Text.RegularExpressions.Regex.Split(expression, "lessthan|<", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();

            string operatorString = string.Empty;
            string leftString = string.Empty;
            int leftInt = 0;
            double leftDouble = 0.0;
            string rightString = string.Empty;
            int rightInt = 0;
            double rightDouble = 0.0;

            if (expressionList_CompareA.Count > 1)
            {
                leftString = expressionList_CompareA[0].Replace("\"", string.Empty).Trim();
                rightString = expressionList_CompareA[1].Replace("\"", string.Empty).Trim();
                operatorString = "GreaterThanOrEqual";
            }
            else if (expressionList_CompareB.Count > 1)
            {
                leftString = expressionList_CompareB[0].Replace("\"", string.Empty).Trim();
                rightString = expressionList_CompareB[1].Replace("\"", string.Empty).Trim();
                operatorString = "LessThanOrEqual";
            }
            else if (expressionList_CompareC.Count > 1)
            {
                leftString = expressionList_CompareC[0].Replace("\"", string.Empty).Trim();
                rightString = expressionList_CompareC[1].Replace("\"", string.Empty).Trim();
                operatorString = "NotEqual";
            }
            else
            {
                if (expression.Contains("=>"))
                {                    
                    File.AppendAllText(errFilePath, "Invalid Operator => in Row :" + rowNumber + "; Value : " + expression + "\n");
                    return new RuleExpressions();
                }
                else if (expression.Contains("=<"))
                {                    
                    File.AppendAllText(errFilePath, "Invalid Operator =< in Row : " + rowNumber + "; Value : " + expression + "\n");
                    return new RuleExpressions();
                }
                else if (expression.Contains("=!"))
                {                    
                    File.AppendAllText(errFilePath, "Invalid Operator =! in Row : " + rowNumber + "; Value : " + expression + "\n");
                    return new RuleExpressions();
                }
                else if (expressionList_CompareD.Count > 1)
                {
                    leftString = expressionList_CompareD[0].Replace("\"", string.Empty).Trim();
                    rightString = expressionList_CompareD[1].Replace("\"", string.Empty).Trim();
                    operatorString = "Equal";
                }
                else if (expressionList_CompareE.Count > 1)
                {
                    leftString = expressionList_CompareE[0].Replace("\"", string.Empty).Trim();
                    rightString = expressionList_CompareE[1].Replace("\"", string.Empty).Trim();
                    operatorString = "GreaterThan";
                }
                else if (expressionList_CompareF.Count > 1)
                {
                    leftString = expressionList_CompareF[0].Replace("\"", string.Empty).Trim();
                    rightString = expressionList_CompareF[1].Replace("\"", string.Empty).Trim();
                    operatorString = "LessThan";
                }
            }

            RuleExpressions rule = new RuleExpressions()
            {
                Operator = operatorString
            };

            if (int.TryParse(leftString, out leftInt))
            {
                rule.Left = leftInt;
            }
            else if (double.TryParse(leftString, out leftDouble))
            {
                rule.Left = leftDouble;
            }
            else
            {
                rule.Left = leftString;
            }

            if (int.TryParse(rightString, out rightInt))
            {
                rule.Right = rightInt;
            }
            else if (double.TryParse(rightString, out rightDouble))
            {
                rule.Right = rightDouble;
            }
            else
            {
                rule.Right = rightString;
            }

            if (!string.IsNullOrWhiteSpace(expression) && string.IsNullOrWhiteSpace(operatorString))
            {
                File.AppendAllText(errFilePath, "Invalid operator in Row : " + rowNumber + "; Value : " + expression + "\n");
                return rule;
            }
            /*if (!string.IsNullOrWhiteSpace(expression) && !string.IsNullOrWhiteSpace(leftString))
            {
                var errRuleProperty = IsRulePropertyExists(leftString);
                if (!string.IsNullOrWhiteSpace(errRuleProperty))
                {
                    if (errRuleProperty == "Error in RuleProperty")
                    {
                        File.AppendAllText(errFilePath, "Either request_data or calculated_parameter_list needs to be added for RuleProperty in Row : " + rowNumber + "; Value : " + expression + "\n");
                        return rule;
                    }
                    if (errRuleProperty == "RuleProperty not configured")
                    {
                        File.AppendAllText(errFilePath, "RuleProperty is not configured in Row : " + rowNumber + "; Value : " + expression + "\n");
                        return rule;
                    }
                }               
            }*/
            
            if (leftString.Equals("request_data[salary]", StringComparison.OrdinalIgnoreCase) && rightString.Equals("varies", StringComparison.OrdinalIgnoreCase) && !operatorString.Equals("equal", StringComparison.OrdinalIgnoreCase))
            {
                File.AppendAllText(errFilePath, string.Format(operatorString + " between " + leftString + " and " + rightString + " is not valid in Row : ") + rowNumber + "\n");
            }

            //RulePayLoadValidationForErrorFile.LeftRightCheck_Validate(leftString, rightString, errFilePath, rowNumber);
            if ((leftString.EndsWith("Age]") || leftString.Equals("calculated_parameter_list[ConfinementDuration]", StringComparison.OrdinalIgnoreCase)))
            {
                var isNumeric = int.TryParse(rightString, out int n);
                if (!isNumeric)
                {
                    File.AppendAllText(errFilePath, "Right Value is Invalid for Left Value in Row :" + rowNumber + "; Right Value : " + rightString + " Left Value : " + leftString.ToString() + "\n");
                }
            }

          /*  if (!string.IsNullOrWhiteSpace(expression) && !string.IsNullOrWhiteSpace(rightString))
            {
                if (rightString.Contains("and ", StringComparison.OrdinalIgnoreCase) || rightString.Contains("or ", StringComparison.OrdinalIgnoreCase))
                {
                    var firstWord = string.Join(" ", rightString.ToLower().Split("and ").Skip(1).Take(1));
                    var rulePropertyExists = string.Empty;
                    if (!string.IsNullOrWhiteSpace(firstWord))
                    {
                        //rulePropertyExists = IsRulePropertyExists(firstWord);
                        //if (!string.IsNullOrWhiteSpace(rulePropertyExists) && rulePropertyExists.Equals("No Error"))
                        //{
                            File.AppendAllText(errFilePath, "Right value concatenated with Property in Row : " + rowNumber + "; Value : " + rightString + "\n");
                            return rule;
                        //}
                    }
                    firstWord = string.Join(" ", rightString.ToLower().Split("or ").Skip(1).Take(1));
                    //rulePropertyExists = IsRulePropertyExists(firstWord);
                    //if (!string.IsNullOrWhiteSpace(rulePropertyExists) && rulePropertyExists.Equals("No Error"))
                    //{
                        File.AppendAllText(errFilePath, "Right value concatenated with Property in Row : " + rowNumber + "; Value : " + rightString + "\n");
                        return rule;
                    //}
                }
            }*/
          
            return rule;
        }

        private string IsRulePropertyExists(string ruleProperty)
        {
            var lstRuleProperty = ValidateRulePropertyManager.RulePropertyMappings.FindAll(keyword => keyword.RuleMappings.Any(a => (a.RuleProperty.ToLower() == ruleProperty.ToLower() || a.MappedToRuleProperty.ToLower() == ruleProperty.ToLower()))).FirstOrDefault();
            if (lstRuleProperty != null && lstRuleProperty.RuleMappings != null && lstRuleProperty.RuleMappings.Count > 0)
            {
                foreach (var rule in lstRuleProperty.RuleMappings)
                {
                    if (rule.RuleProperty.Equals(rule.MappedToRuleProperty, StringComparison.OrdinalIgnoreCase) && rule.RuleProperty.Equals(ruleProperty, StringComparison.OrdinalIgnoreCase))
                    {
                        return "No Error";          
                    }
                    if (!rule.RuleProperty.Equals(rule.MappedToRuleProperty, StringComparison.OrdinalIgnoreCase) && rule.MappedToRuleProperty.Equals(ruleProperty, StringComparison.OrdinalIgnoreCase))
                    {
                        return "No Error";
                    }
                }
                return "Error in RuleProperty"  ;
            }
            return "RuleProperty not configured";
        }

        private RuleExpressions BuildExpression_Operator(string expresssion)
        {
            List<string> expressionList_CompareA = System.Text.RegularExpressions.Regex.Split(expresssion, "greaterthanorequal|>=", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareB = System.Text.RegularExpressions.Regex.Split(expresssion, "lessthanorequal|<=", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareC = System.Text.RegularExpressions.Regex.Split(expresssion, "notequal|!=", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareD = System.Text.RegularExpressions.Regex.Split(expresssion, "equal|=", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareE = System.Text.RegularExpressions.Regex.Split(expresssion, "greaterthan|>", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();
            List<string> expressionList_CompareF = System.Text.RegularExpressions.Regex.Split(expresssion, "lessthan|<", System.Text.RegularExpressions.RegexOptions.IgnoreCase).ToList();

            string operatorString = string.Empty;

            string leftString = string.Empty;
            int leftInt = 0;
            double leftDouble = 0.0;

            string rightString = string.Empty;
            int rightInt = 0;
            double rightDouble = 0.0;

            if (expressionList_CompareA.Count > 1)
            {
                leftString = expressionList_CompareA[0].Replace("\"", string.Empty).Trim();
                rightString = expressionList_CompareA[1].Replace("\"", string.Empty).Trim();
                operatorString = "GreaterThanOrEqual";
            }
            else if (expressionList_CompareB.Count > 1)
            {
                leftString = expressionList_CompareB[0].Replace("\"", string.Empty).Trim();
                rightString = expressionList_CompareB[1].Replace("\"", string.Empty).Trim();
                operatorString = "LessThanOrEqual";
            }
            else if (expressionList_CompareC.Count > 1)
            {
                leftString = expressionList_CompareC[0].Replace("\"", string.Empty).Trim();
                rightString = expressionList_CompareC[1].Replace("\"", string.Empty).Trim();
                operatorString = "NotEqual";
            }
            else
            {
                if (expresssion.Contains("=>"))
                {
                    throw new Exception("Invalid Operator =>");
                }
                else if (expresssion.Contains("=<"))
                {
                    throw new Exception("Invalid Operator =<");
                }
                else if (expresssion.Contains("=!"))
                {
                    throw new Exception("Invalid Operator =!");
                }
                else if (expressionList_CompareD.Count > 1)
                {
                    leftString = expressionList_CompareD[0].Replace("\"", string.Empty).Trim();
                    rightString = expressionList_CompareD[1].Replace("\"", string.Empty).Trim();
                    operatorString = "Equal";
                }
                else if (expressionList_CompareE.Count > 1)
                {
                    leftString = expressionList_CompareE[0].Replace("\"", string.Empty).Trim();
                    rightString = expressionList_CompareE[1].Replace("\"", string.Empty).Trim();
                    operatorString = "GreaterThan";
                }
                else if (expressionList_CompareF.Count > 1)
                {
                    leftString = expressionList_CompareF[0].Replace("\"", string.Empty).Trim();
                    rightString = expressionList_CompareF[1].Replace("\"", string.Empty).Trim();
                    operatorString = "LessThan";
                }               
            }

            RuleExpressions rule = new RuleExpressions()
            {
                Operator = operatorString
            };

            if (int.TryParse(leftString, out leftInt))
            {
                rule.Left = leftInt;
            }
            else if (double.TryParse(leftString, out leftDouble))
            {
                rule.Left = leftDouble;
            }
            else
            {
                rule.Left = leftString;
            }

            if (int.TryParse(rightString, out rightInt))
            {
                rule.Right = rightInt;
            }
            else if (double.TryParse(rightString, out rightDouble))
            {
                rule.Right = rightDouble;
            }
            else
            {
                rule.Right = rightString;
            }

            return rule;
        }
    }

    public class ChargeSeverityValidation
    {
        private int count = 0;
        private  CreateRulesRequest rulesRequest = new CreateRulesRequest();
        private readonly CreateRulesRequest rulesRequestMain = new CreateRulesRequest();
        private int countCopyCollection = 0;
        private int countoriginalCollection = 0;

        public CreateRulesRequest ListOfChargeSeverity(CreateRulesRequest createRulesRequest,Boolean isSimpleExpression)
        {
            rulesRequest.Rules = new List<CreateRuleParameters>();
            var newRuleCollection = new List<CreateRuleParameters>(createRulesRequest.Rules);

            newRuleCollection.ForEach((rules =>
            {
                TraverseJsonElement(rules.Expressions, rules, createRulesRequest);
                count++;
            }));

            createRulesRequest.Rules.AddRange(rulesRequest.Rules);
            return createRulesRequest;
        }
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
                        // if (leftvalue == "ChargeSeverity")
                        //    GetChargeSeverityDetails(rightvalue, createRuleParameters, createRulesRequest);
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

                        /* if (leftvalue == "ChargeSeverity")
                         {
                             GetChargeSeverityDetails(rightExpr.Right.ToString(), createRuleParameters, createRulesRequest);
                         }*/
                    }
                    TraverseJsonElement(leftExpr, createRuleParameters, createRulesRequest);
                    TraverseJsonElement(rightExpr, createRuleParameters, createRulesRequest);
                }

            }
        }

        private RuleExpressions ReplaceChargeSeverity(RuleExpressions Root, string ChargeLevelMapping, string ChargeDetails)
        {
            RuleExpressions root = (RuleExpressions)Root.Clone();
            if (Root.Operator != null)
            {
                var leftElement = (JsonElement)root.Left;
                var rightElement = (JsonElement)root.Right;

                if (leftElement.ValueKind != JsonValueKind.String)
                {
                    if (leftElement.GetRawText().Contains("ChargeSeverity"))
                    {
                        root.Left = JsonSerializer.Deserialize<JsonElement>((leftElement.GetRawText().Replace(ChargeDetails, ChargeLevelMapping)), new JsonSerializerOptions
                        {
                            MaxDepth = 512
                        });
                    }
                }
                if (rightElement.ValueKind != JsonValueKind.String)
                {
                    if (rightElement.GetRawText().Contains("ChargeSeverity"))
                    {
                        root.Right = JsonSerializer.Deserialize<JsonElement>((rightElement.GetRawText().Replace(ChargeDetails, ChargeLevelMapping)), new JsonSerializerOptions
                        {
                            MaxDepth = 512
                        });
                    }
                }
            }

            return root;
        }

        private CreateRulesRequest GetChargeSeverityDetails(string chargeDetails, CreateRuleParameters createRuleParameters, CreateRulesRequest createRulesRequest)
        {
            List<CreateRuleParameters> collectionRuleParamters = new List<CreateRuleParameters>();
            var mapping = GetChargeMapping(chargeDetails);

            if (mapping.Count > 0)
            {
                countCopyCollection++;
                for (int i = 0; i < mapping.Count; i++)
                {
                    CreateRuleParameters lstRuleParameters = new CreateRuleParameters();
                    lstRuleParameters.Expressions = new RuleExpressions();
                    lstRuleParameters.Expressions = createRuleParameters.Expressions;
                    lstRuleParameters.Expressions = ReplaceChargeSeverity(lstRuleParameters.Expressions, mapping[i].Description, chargeDetails);
                    lstRuleParameters.ExpressionString = createRuleParameters.ExpressionString.Replace(chargeDetails, mapping[i].Description);
                    lstRuleParameters.RuleType = createRuleParameters.RuleType;
                    lstRuleParameters.Reportable = createRuleParameters.Reportable;
                    lstRuleParameters.Score = createRuleParameters.Score;
                    lstRuleParameters.Id = createRuleParameters.Id;
                    lstRuleParameters.Points = createRuleParameters.Points;
                    lstRuleParameters.Label = createRuleParameters.Label;
                    lstRuleParameters.LabelRank = createRuleParameters.LabelRank;
                    lstRuleParameters.Frequency = createRuleParameters.Frequency;
                    lstRuleParameters.Country = createRuleParameters.Country;
                    lstRuleParameters.State = createRuleParameters.State;
                    collectionRuleParamters.Add(lstRuleParameters);
                }
               
               // createRulesRequest.Rules.RemoveAt(countoriginalCollection);
                 rulesRequest.Rules.AddRange(collectionRuleParamters);
            }
            else
            {
                countoriginalCollection++;
                countCopyCollection++;
            }

            return rulesRequest;
        }

        private static void NullCheck(RuleExpressions expressions)
        {
            if (expressions.Left == null)
            {
                throw new ApplicationException("Left Value null");
            }
            if (expressions.Right == null)
            {
                throw new ApplicationException("Right Value null is Invalid");
            }
        }

        public List<ChargeLevelMapping> GetChargeMapping(string chargeLevelMappingName)
        {
            return ChargeLevelMappingManager.Mappings.FindAll(m =>
                m.Mapping.Equals(chargeLevelMappingName, StringComparison.OrdinalIgnoreCase));
        }
    }

    public class ChargeLevelMapping
    {
        [BsonElement("_id")]
        public ObjectId Id { get; set; }

        [BsonElement("charge_level_code")]
        public string Code { get; set; }

        [BsonElement("charge_level_mapping")]
        public string Mapping { get; set; }

        [BsonElement("charge_level_description")]
        public string Description { get; set; }
    }

    public class ChargeLevelMappingManager
    {
        public static List<ChargeLevelMapping> Mappings { get; private set; }
        private static MongoDb db;
        public static void Load(int index)
        {
            if (db != null)
            {
                return;
            }
            db = new MongoDb();
            db.Init(index);

            Mappings = db.ChargeLevelMappings.Find(_ => true).ToList();
        }

        public static string GetExistingModelId(string hashCode)
        {
            var existing = db.Rules.Find(r => r.HashCode == hashCode).FirstOrDefault();
            return existing?.ModelId;
        }

        public static void Save(List<RuleModel> rules)
        {
            BsonDefaults.MaxSerializationDepth = 512;
            db.Rules.InsertMany(rules);
           // db.RulesHistory.InsertMany(rules);
        }        
    }    

    public class RulePayLoadValidationForErrorFile
    {       
        public static bool IsValid_Validate(CreateRuleParameters rule, string errFilePath, int rowNumber)
        {
            if (rule.Expressions.Operator == null || rule.RuleType == null)
            {               
                File.AppendAllText(errFilePath, "Rule Type Parameter value in " + rowNumber + "\n");
                return false;
            }
            if (!IsValidRuleType_Validate(rule.RuleType))
            {
                File.AppendAllText(errFilePath, string.Format(" Rule Type {0}", rule.RuleType) + rowNumber + "\n");
                return false;
            }
            NullValueCheck_Validate(rule.Expressions, errFilePath, rowNumber);
            //var leftElement = (JsonElement)rule.Expressions.Left;
            //var rightElement = (JsonElement)rule.Expressions.Right;
            var leftElement = rule.Expressions.Left;
            var rightElement = rule.Expressions.Right;

           // if (leftElement.ValueKind == JsonValueKind.String || leftElement.ValueKind == JsonValueKind.Number)
           // {                
                string leftvalue = leftElement.ToString();
                //if (!Enum.IsDefined(typeof(RelationalOperator), rule.Expressions.Operator))
                //{
                //    File.AppendAllText(errFilePath, string.Format(" Relational operator {0} in", rule.Expressions.Operator.ToString()) + rowNumber + "\n");
                //    return false;
                //}
                string rightvalue = rightElement.ToString();
                LeftRightCheck_Validate(leftvalue, rightvalue, errFilePath, rowNumber);
                if ((leftvalue.EndsWith("Age") || leftvalue == "ConfinementDuration"))
                {
                    var isNumeric = int.TryParse(rightvalue.ToString(), out int n);
                    if (!isNumeric)
                    {
                        File.AppendAllText(errFilePath, string.Format(" Right Value is Invalid for Left Value {0} in ", leftvalue.ToString()) + rowNumber + "\n");
                        return false;
                    }                    
                }
           // SalaryValueCheck_Validate(leftvalue, rightvalue.ToString(), rightExpr.Operator.ToString(), errFilePath, rowNumber);
            // }
            //else
            //{
            //  Preoder_Validate(rule.Expressions, errFilePath, rowNumber);
            //}
            return true;
        }

        private static void Preoder_Validate(RuleExpressions Root, string errFilePath, int rowNumber)
        {
            if (Root.Operator != null)
            {
                var leftElement = (JsonElement)Root.Left;
                var rightElement = (JsonElement)Root.Right;
                RuleExpressions leftExpr = new RuleExpressions(), rightExpr = new RuleExpressions();
                if (leftElement.ValueKind != JsonValueKind.String)
                {
                    if (Root.Operator == null)
                    {                        
                        File.AppendAllText(errFilePath, "Logical Operator Value null in " + rowNumber + "\n"); 
                    }
                    if (!Enum.IsDefined(typeof(LogicalOperator), Root.Operator))
                    {
                        File.AppendAllText(errFilePath, string.Format("Logical Operator {0}", Root.Operator.ToString()) + rowNumber + "\n");
                    }
                    string leftvalue = string.Empty, rightvalue;
                    leftExpr = JsonSerializer.Deserialize<RuleExpressions>(leftElement.GetRawText(), new JsonSerializerOptions
                    {
                        MaxDepth = 512
                    });
                    NullValueCheck_Validate(leftExpr, errFilePath, rowNumber);
                    var LeftleftElement = (JsonElement)leftExpr.Left;
                    var LeftrightElement = (JsonElement)leftExpr.Right;
                    if (LeftleftElement.ValueKind == JsonValueKind.String)
                    {
                        if (leftExpr.Operator == null)
                        {
                            File.AppendAllText(errFilePath, "Relational Operator Value null in " + rowNumber + "\n");
                        }
                        if (!Enum.IsDefined(typeof(RelationalOperator), leftExpr.Operator))
                        {
                            File.AppendAllText(errFilePath, string.Format("Relational Operator {0}", leftExpr.Operator.ToString()) + rowNumber + "\n");
                        }
                        leftvalue = leftExpr.Left.ToString();                       
                    }
                    if (LeftrightElement.ValueKind == JsonValueKind.String || LeftrightElement.ValueKind == JsonValueKind.Number)
                    {
                        rightvalue = leftExpr.Right.ToString();
                        LeftRightCheck_Validate(leftvalue, rightvalue, errFilePath, rowNumber);
                        if ((leftvalue.EndsWith("Age") || leftvalue == "ConfinementDuration") && (LeftrightElement.ValueKind != JsonValueKind.Number))
                        {
                            File.AppendAllText(errFilePath, "Right Value null is Invalid for Left Value {0} in Row :" + rowNumber + "Value : " + leftvalue.ToString() + "\n");
                        }                        
                    }
                }
                if (rightElement.ValueKind != JsonValueKind.String && rightElement.ValueKind != JsonValueKind.Number)
                {
                    if (Root.Operator == null)
                    {
                        File.AppendAllText(errFilePath, "Logical Operator Value null in " + rowNumber + "\n");
                    }
                    if (!Enum.IsDefined(typeof(LogicalOperator), Root.Operator))
                    {
                        File.AppendAllText(errFilePath, string.Format("Logical Operator {0}", Root.Operator.ToString()) + rowNumber + "\n");
                    }
                    string leftvalue = string.Empty, rightvalue;
                    rightExpr = JsonSerializer.Deserialize<RuleExpressions>(rightElement.GetRawText(), new JsonSerializerOptions
                    {
                        MaxDepth = 512
                    });
                    NullValueCheck_Validate(rightExpr, errFilePath, rowNumber);

                    var rightleftElement = (JsonElement)rightExpr.Left;
                    var rightrightElement = (JsonElement)rightExpr.Right;
                    if (rightleftElement.ValueKind == JsonValueKind.String)
                    {
                        if (rightExpr.Operator == null)
                        {
                            File.AppendAllText(errFilePath, "Relational Operator Value null " + rowNumber + "\n");
                        }
                        if (!Enum.IsDefined(typeof(RelationalOperator), rightExpr.Operator))
                        {
                            File.AppendAllText(errFilePath, string.Format("Relational Operator {0}", rightExpr.Operator.ToString()) + rowNumber + "\n");
                        }
                        leftvalue = rightExpr.Left.ToString();
                    }

                    if (rightrightElement.ValueKind == JsonValueKind.String)
                    {
                        rightvalue = rightExpr.Right.ToString();
                        LeftRightCheck_Validate(leftvalue, rightvalue, errFilePath, rowNumber);
                        if ((leftvalue.EndsWith("Age") || leftvalue == "ConfinementDuration") && (rightrightElement.ValueKind != JsonValueKind.Number))
                        {
                            File.AppendAllText(errFilePath, string.Format("Right Value is Invalid for Left Value {0}", leftvalue.ToString()) + rowNumber + "\n");
                        }
                    }
                }
                Preoder_Validate(leftExpr, errFilePath, rowNumber);
                Preoder_Validate(rightExpr, errFilePath, rowNumber);
            }            
        }

        private static bool IsValidRuleType_Validate(string ruleType)
        {
            var ruleTypeValue = new[] { "FCRA", "CLIENT", "ADJUDICATION", "FCRA-MISC", "DOJSON" };
            if (ruleTypeValue.Contains(ruleType, StringComparer.OrdinalIgnoreCase))
            {
                return true;
            }
            return false;
        }        

        private static void NullValueCheck_Validate(RuleExpressions expressions, string errFilePath, int rowNumber)
        {
            if (expressions.Right == null)
            {                
                File.AppendAllText(errFilePath, "Right Value null is Invalid in Row :" + rowNumber + "\n");
            }
        }

        public static void LeftRightCheck_Validate(string leftvalue, string rightvalue, string errFilePath, int rowNumber)
        {
            if (leftvalue.EndsWith("Outcome", StringComparison.OrdinalIgnoreCase)  && (!CompareOutcomeValue(rightvalue)))
            {                
                File.AppendAllText(errFilePath, "Right Value is Invalid for Left Value in Row : " + rowNumber + " Right Value : " + rightvalue + "; Left Value : " + leftvalue + "\n");
            }
            if ((leftvalue.Equals("calculated_parameter_list[WarrantStatus]", StringComparison.OrdinalIgnoreCase) || leftvalue.Equals("calculated_parameter_list[ProbationStatus]", StringComparison.OrdinalIgnoreCase)) && (!Enum.IsDefined(typeof(ChargeStatus), rightvalue)))
            {
                File.AppendAllText(errFilePath, "Right Value is Invalid for Left Value in Row : " + rowNumber + " Right Value : " + rightvalue + "; Left Value : " + leftvalue + "\n");
            }
            if (leftvalue.Equals("ChargeSeverity", StringComparison.OrdinalIgnoreCase))
            {
                if (IsChargeMappingExists(rightvalue) == false)
                    File.AppendAllText(errFilePath, "Right Value is Invalid for Left Value in Row : " + rowNumber + " Right Value : " + rightvalue + "; Left Value : " + leftvalue + "\n");
            }
        } 

        public static bool IsChargeMappingExists(string chargeLevelDescription)
        {
            return ChargeLevelMappingManager.Mappings.Exists(m =>
                m.Description.Equals(chargeLevelDescription, StringComparison.CurrentCultureIgnoreCase));
        }

        enum RelationalOperator
        {
            Equal,
            NotEqual,
            GreaterThan,
            LessThan,
            LessThanOrEqual,
            GreaterThanOrEqual,
        }

        enum LogicalOperator
        {
            And,
            Or,
        }        

        enum ChargeOutcome
        {
            Active,
            Conviction,
            AlternateDisposition,
            NonConviction,
            Unknown
        }
        enum ChargeStatus
        {
            Active,
            InActive,
            Unknown
        }

        private static bool CompareOutcomeValue(string rightValue)
        {
            var outcomeValues = new[] { "Active", "Conviction", "Alternate Disposition", "Non-Conviction", "Unknown" };
            if (outcomeValues.Contains(rightValue, StringComparer.OrdinalIgnoreCase))
            {
                return true;
            }
            return false;
        }
    }
    public class RulePayLoadValidation
    {
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

        private static void Preoder(RuleExpressions Root)
        {
            if (Root.Operator != null)
            {
                var leftElement = (JsonElement)Root.Left;
                var rightElement = (JsonElement)Root.Right;
                RuleExpressions leftExpr = new RuleExpressions(), rightExpr = new RuleExpressions();
                if (leftElement.ValueKind != JsonValueKind.String)
                {
                    if (Root.Operator == null)
                    {
                        throw new ApplicationException("Logical Operator Value null");
                    }
                    if (!Enum.IsDefined(typeof(LogicalOperator), Root.Operator))
                    {
                        throw new ApplicationException(string.Format("Logical Operator {0}", Root.Operator.ToString()));
                    }
                    string leftvalue = string.Empty, rightvalue;
                    leftExpr = JsonSerializer.Deserialize<RuleExpressions>(leftElement.GetRawText(), new JsonSerializerOptions
                    {
                        MaxDepth = 512
                    });
                    NullValueCheck(leftExpr);
                    var LeftleftElement = (JsonElement)leftExpr.Left;
                    var LeftrightElement = (JsonElement)leftExpr.Right;
                    if (LeftleftElement.ValueKind == JsonValueKind.String)
                    {
                        if (leftExpr.Operator == null)
                        {
                            throw new ApplicationException("Relational Operator Value null");
                        }
                        if (!Enum.IsDefined(typeof(RelationalOperator), leftExpr.Operator))
                        {
                            throw new ApplicationException(string.Format("Relational Operator {0}", leftExpr.Operator.ToString()));
                        }
                        leftvalue = leftExpr.Left.ToString();

                    }
                    if (LeftrightElement.ValueKind == JsonValueKind.String || LeftrightElement.ValueKind == JsonValueKind.Number)
                    {
                        rightvalue = leftExpr.Right.ToString();
                        LeftRightCheck(leftvalue, rightvalue);
                        if ((leftvalue.EndsWith("Age") || leftvalue == "ConfinementDuration") && (LeftrightElement.ValueKind != JsonValueKind.Number))
                        {
                            throw new ApplicationException(string.Format("Right Value is Invalid for Left Value {0}", leftvalue.ToString()));
                        }
                    }
                }
                if (rightElement.ValueKind != JsonValueKind.String && rightElement.ValueKind != JsonValueKind.Number)
                {
                    if (Root.Operator == null)
                    {
                        throw new ApplicationException("Logical Operator Value null");
                    }
                    if (!Enum.IsDefined(typeof(LogicalOperator), Root.Operator))
                    {
                        throw new ApplicationException(string.Format("Logical Operator {0}", Root.Operator.ToString()));
                    }
                    string leftvalue = string.Empty, rightvalue;
                    rightExpr = JsonSerializer.Deserialize<RuleExpressions>(rightElement.GetRawText(), new JsonSerializerOptions
                    {
                        MaxDepth = 512
                    });
                    NullValueCheck(rightExpr);
                    var rightleftElement = (JsonElement)rightExpr.Left;
                    var rightrightElement = (JsonElement)rightExpr.Right;
                    if (rightleftElement.ValueKind == JsonValueKind.String)
                    {
                        if (rightExpr.Operator == null)
                        {
                            throw new ApplicationException("Relational Operator Value null");
                        }
                        if (!Enum.IsDefined(typeof(RelationalOperator), rightExpr.Operator))
                        {
                            throw new ApplicationException(string.Format("Relational Operator {0}", rightExpr.Operator.ToString()));
                        }
                        leftvalue = rightExpr.Left.ToString();

                    }

                    if (rightrightElement.ValueKind == JsonValueKind.String)
                    {
                        rightvalue = rightExpr.Right.ToString();
                        LeftRightCheck(leftvalue, rightvalue);
                        if ((leftvalue.EndsWith("Age") || leftvalue == "ConfinementDuration") && (rightrightElement.ValueKind != JsonValueKind.Number))
                        {
                            throw new ApplicationException(string.Format("Right Value is Invalid for Left Value {0}", leftvalue.ToString()));
                        }
                    }
                }
                Preoder(leftExpr);
                Preoder(rightExpr);
            }
        }

        private static void LeftRightCheck(string leftvalue, string rightvalue)
        {
            string rightvalueException = string.Format("Right Value is Invalid for Left Value {0}", leftvalue.ToString());

            if (leftvalue == "ChargeOutcome" && (!CompareOutcomeValue(rightvalue)))
            {
                throw new ApplicationException(rightvalueException);
            }
            if ((leftvalue == "WarrantStatus" || leftvalue == "ProbationStatus") && (!Enum.IsDefined(typeof(ChargeStatus), rightvalue)))
            {
                throw new ApplicationException(rightvalueException);
            }
            if ((leftvalue == "ChargeSeverity"))
            {
                if (IsChargeMappingExists(rightvalue) == false)
                    throw new ApplicationException(rightvalueException);
            }
        }

        public static bool IsChargeMappingExists(string chargeLevelDescription)
        {
            return ChargeLevelMappingManager.Mappings.Exists(m =>
                m.Description.Equals(chargeLevelDescription, StringComparison.CurrentCultureIgnoreCase));
        }

        private static void NullValueCheck(RuleExpressions expressions)
        {
            if (expressions.Right == null)
            {
                throw new ApplicationException("Right Value null is Invalid");
            }
        }

        private static bool CompareOutcomeValue(string rightValue)
        {
            var outcomeValues = new[] { "Active", "Conviction", "Alternate Disposition", "Non-Conviction", "Unknown" };
            if (outcomeValues.Contains(rightValue, StringComparer.OrdinalIgnoreCase))
            {
                return true;
            }
            return false;
        }

        private static bool IsValidRuleType(string ruleType)
        {
            var ruleTypeValue = new[] { "FCRA", "CLIENT", "ADJUDICATION", "FCRA-MISC", "DOJSON","MVR" };
            if (ruleTypeValue.Contains(ruleType, StringComparer.OrdinalIgnoreCase))
            {
                return true;
            }
            return false;
        }

        enum LogicalOperator
        {
            And,
            Or,
        }
        enum RelationalOperator
        {
            Equal,
            NotEqual,
            GreaterThan,
            LessThan,
            LessThanOrEqual,
            GreaterThanOrEqual,
        }

        enum ChargeOutcome
        {
            Active,
            Conviction,
            AlternateDisposition,
            NonConviction,
            Unknown
        }
        enum ChargeStatus
        {
            Active,
            InActive,
            Unknown
        }
    }

    public class RuleComparer : IComparer<CreateRuleParameters>
    {
        public int Compare(CreateRuleParameters x, CreateRuleParameters y)
        {
            if (x == null) throw new ArgumentNullException(nameof(x));
            if (y == null) throw new ArgumentNullException(nameof(y));

            if (x.Frequency.CompareTo(y.Frequency) != 0)
            {
                return x.Frequency.CompareTo(y.Frequency);
            }

            if (x.Points.CompareTo(y.Points) != 0)
            {
                return x.Points.CompareTo(y.Points);
            }

            if (String.Compare(x.Country, y.Country, StringComparison.Ordinal) != 0)
            {
                return String.Compare(x.Country, y.Country, StringComparison.Ordinal);
            }

            var expressionResult = CompareExpressionsTo(x.Expressions, y.Expressions);
            if (expressionResult != 0)
            {
                return expressionResult;
            }

            if (String.Compare(x.Label, y.Label, StringComparison.Ordinal) != 0)
            {
                return String.Compare(x.Label, y.Label, StringComparison.Ordinal);
            }

            if (x.LabelRank.CompareTo(y.LabelRank) != 0)
            {
                return x.LabelRank.CompareTo(y.LabelRank);
            }

            if (x.Reportable.CompareTo(y.Reportable) != 0)
            {
                return x.Reportable.CompareTo(y.Reportable);
            }

            if (String.Compare(x.Score, y.Score, StringComparison.Ordinal) != 0)
            {
                return String.Compare(x.Score, y.Score, StringComparison.Ordinal);
            }

            if (String.Compare(x.State, y.State, StringComparison.Ordinal) != 0)
            {
                return String.Compare(x.State, y.State, StringComparison.Ordinal);
            }

            if (String.Compare(x.RuleType, y.RuleType, StringComparison.Ordinal) != 0)
            {
                return String.Compare(x.RuleType, y.RuleType, StringComparison.Ordinal);
            }

            return 0;
        }

        private int CompareExpressionsTo(RuleExpressions x, RuleExpressions y)
        {
            if (String.Compare(x.Operator, y.Operator, StringComparison.Ordinal) != 0)
            {
                return String.Compare(x.Operator, y.Operator, StringComparison.Ordinal);
            }

          /*  if (x.LeftRuleExpressions != null && y.LeftRuleExpressions != null)
            {
                var result = CompareExpressionsTo(x.LeftRuleExpressions, y.LeftRuleExpressions);

                if (result != 0)
                {
                    return result;
                }
            }

            if (x.RightRuleExpressions != null && y.RightRuleExpressions != null)
            {
                var result = CompareExpressionsTo(x.RightRuleExpressions, y.RightRuleExpressions);

                if (result != 0)
                {
                    return result;
                }
            }*/

            var xLeftElement = (JsonElement)x.Left;
            var xRightElement = (JsonElement)x.Right;
            var yLeftElement = (JsonElement)y.Left;
            var yRightElement = (JsonElement)y.Right;

            var leftResult = String.Compare(xLeftElement.GetRawText(), yLeftElement.GetRawText(),
                StringComparison.Ordinal);

            if (leftResult != 0)
            {
                return leftResult;
            }

            var rightResult = String.Compare(xRightElement.GetRawText(), yRightElement.GetRawText(),
                StringComparison.Ordinal);

            if (rightResult != 0)
            {
                return rightResult;
            }

            return 0;
        }
    }

    public static class HashCodeGenerate
    {
        public static string ToSha512(List<CreateRuleParameters> ruleParameter)
        {

            string serializeData = JsonSerializer.Serialize(ruleParameter);
            SHA512 shaM = new SHA512Managed();
            return Convert.ToBase64String(shaM.ComputeHash(Encoding.UTF8.GetBytes(serializeData.ToUpper())));
        }
    }

    [BsonIgnoreExtraElements]
    public class RuleModel
    {
        [BsonElement("_id")]
        public string Id { get; set; }

        [BsonElement("model_id")]
        public string ModelId { get; set; }

        [BsonElement("hash_code")]
        public string HashCode { get; set; }

        [BsonElement("state")]
        public string State { get; set; }

        [BsonElement("country")]
        public string Country { get; set; }

        [BsonElement("type")]
        public string RuleType { get; set; }

        [BsonElement("reportable")]
        public bool Reportable { get; set; }

        [BsonElement("score")]
        public string Score { get; set; }

        [BsonElement("points")]
        public double Points { get; set; }

        [BsonElement("frequency")]
        public int Frequency { get; set; }

        [BsonElement("label")]
        public string Label { get; set; }

        [BsonElement("label_rank")]
        public int LabelRank { get; set; }


        [BsonElement("expressions")]
        [JsonPropertyName("expressions")]
        public BsonDocument Expressions { get; set; }
        //public BsonDocument Expressions { get; set; }
        // [BsonIgnore]
        [BsonElement("complexExpressions")]
        public string ExpressionString { get; set; }
    }
}
