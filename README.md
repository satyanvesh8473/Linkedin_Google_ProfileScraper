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
                // RuleExpressions expressions = BuildExpression(columnList[9], "&&");
                RuleExpressions expressions = null;
                string complexExpressionValue = columnList[9].Replace("\"", "");
                string expression = columnList[9];
                // Fix Excel double quotes
                expression = expression.Replace("\"\"", "\"");
                // Normalize smart quotes
                expression = expression
                .Replace("“", "\"")
                .Replace("”", "\"");
                // Remove outer wrapping quotes
                if (expression.StartsWith("\"") && expression.EndsWith("\""))
                {
                expression = expression.Substring(1, expression.Length - 2);
                }
                // DEBUG RAW EXPRESSION
                // File.AppendAllText(@"C:\Satya\Rules_upload\raw_expression.txt",
                // "RAW:\n" + expression + "\n\n");
                // File.AppendAllText(@"C:\Satya\Rules_upload\raw_expression.txt",
                // "CHARS:\n");
                // foreach (char ch in expression)
                // {
                // File.AppendAllText(@"C:\Satya\Rules_upload\raw_expression.txt",
                //     $"{ch}  (ASCII: {(int)ch})\n");
                // }
                // File.AppendAllText(@"C:\Satya\Rules_upload\raw_expression.txt",
                // "-----------------------------------\n\n");
                var tokenizer = new ExpressionTokenizer();
                var tokens = tokenizer.Tokenize(expression);
                var parser = new ExpressionParser(tokens);
                expressions = parser.Parse();
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
                            ExpressionString = complexExpressionValue
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
