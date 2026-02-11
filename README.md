using System;
using System.Collections.Generic;
using RulesUpload.CreateRules;

public class ExpressionTokenizer
{
   public List<Token> Tokenize(string expression)
   {
       var tokens = new List<Token>();
       int i = 0;
       while (i < expression.Length)
       {
           char c = expression[i];
           // Skip whitespace
           if (char.IsWhiteSpace(c))
           {
               i++;
               continue;
           }
           // Parentheses
           if (c == '(')
           {
               tokens.Add(new Token { Type = TokenType.LParen, Value = "(" });
               i++;
               continue;
           }
           if (c == ')')
           {
               tokens.Add(new Token { Type = TokenType.RParen, Value = ")" });
               i++;
               continue;
           }
           // Logical operators
           if (i + 1 < expression.Length)
           {
               string two = expression.Substring(i, 2);
               if (two == "&&")
               {
                   tokens.Add(new Token { Type = TokenType.And, Value = "&&" });
                   i += 2;
                   continue;
               }
               if (two == "||")
               {
                   tokens.Add(new Token { Type = TokenType.Or, Value = "||" });
                   i += 2;
                   continue;
               }
               if (two == ">=" || two == "<=" || two == "!=")
               {
                   tokens.Add(new Token { Type = TokenType.RelOp, Value = two });
                   i += 2;
                   continue;
               }
           }
           // Single char relational
           if (c == '=' || c == '>' || c == '<')
           {
               tokens.Add(new Token { Type = TokenType.RelOp, Value = c.ToString() });
               i++;
               continue;
           }
           // String literal
           if (c == '"')
           {
               int start = ++i;
               while (i < expression.Length && expression[i] != '"')
                   i++;
               string value = expression.Substring(start, i - start);
               tokens.Add(new Token { Type = TokenType.String, Value = value });
               i++; // skip closing quote
               continue;
           }
           // Number
           if (char.IsDigit(c))
           {
               int start = i;
               while (i < expression.Length &&
                      (char.IsDigit(expression[i]) || expression[i] == '.'))
                   i++;
               tokens.Add(new Token
               {
                   Type = TokenType.Number,
                   Value = expression.Substring(start, i - start)
               });
               continue;
           }
           // Identifier
           if (char.IsLetter(c) || c == '_')
           {
               int start = i;
               while (i < expression.Length &&
                      (char.IsLetterOrDigit(expression[i]) || expression[i] == '_'))
                   i++;
               tokens.Add(new Token
               {
                   Type = TokenType.Identifier,
                   Value = expression.Substring(start, i - start)
               });
               continue;
           }
           throw new ApplicationException($"Unexpected character: {c}");
       }
       tokens.Add(new Token { Type = TokenType.End });
       return tokens;
   }
}
