# Characteristics of `()+?` in Interpreter Mode #
`()+?` can cause regular expressions in interpreter mode, i.e., when using `Regex.Match(input, pattern)` or `new Regex(pattern).Match(input)`, to produce results that differ from those in compiled mode, i.e., when using `new Regex(pattern, RegexOptions.Compiled).Matches(input)`.  
Furthermore, in interpreter mode, if the regular expression contains `()+?` or equivalent sub-expressions, the output can become very peculiar. For example:



    using System.Text.RegularExpressions;
    
    string input = "wtfb";
    string pattern="^(.)+()+?b";
    Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
    Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    
    Console.WriteLine($"Interpreted: {matchInterpreted.Value}");
    Console.WriteLine($"Compiled: {matchCompiled.Value}");
Output: 

    Interpreted: b  
    Compiled: wtfb
Another Example:

    using System.Text.RegularExpressions;
    
    string input = "whatthefuckx";
    string pattern="(?<=(.()+?))x";
    Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
    Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    
    Console.WriteLine($"Interpreted: {matchInterpreted.Value}");
    Console.WriteLine($"Compiled: {matchCompiled.Value}");
Output:

    Interpreted: tthefuckx  
    Compiled: x

For more examples, please check[GitHub Discussion #110976](https://github.com/dotnet/runtime/discussions/110976)

----------
# Bug in Balancing Groups - Part 1 #
In certain situations, when a balancing group has no captures, it should directly return a match failure instead of erroneously throwing an exception.


    using System.Text.RegularExpressions;

    string input = "12";
    string pattern="(.)(?'2-1'(?'-1'.))";
	try
	{
        Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
		Console.WriteLine($"Interpreted: {matchInterpreted.Success}");
	}catch(Exception ex)
	{
		Console.WriteLine($"Interpreted Exception: {ex.Message}");
	}
	
	
	try
	{
        Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
		Console.WriteLine($"Compiled: {matchCompiled.Success}");
	}catch(Exception ex)
	{
		Console.WriteLine($"Compiled Exception: {ex.Message}");
	}
Output:

    Interpreted: False
    Compiled Exception: Index was outside the bounds of the array.

----------
# Bug in Balancing Groups - Part 2 #
The following examples will involve more complex regular expressions.
> For visualizing and debugging complex regular expressions, you can refer to [Regex Visualization and Debugging Tool](https://github.com/longxya/RegexVisual)

In `(?'g1-g2'exp)`, when the content matched by `exp` precedes the latest capture of `g2`, `g1` will exhibit very peculiar 'characteristics'.   

By checking the captures of the group using `Group.Captures`, you will find that the captures appear empty. However, when using `(?(GroupName)yes|no)` for conditional evaluation, you will notice that there actually is a capture. This situation is tentatively referred to as "negative length capture" or "capturing negative length content".  

Firstly, let’s demonstrate this with code output:

    using System.Text.RegularExpressions;
    
    string input = "00123xzacvb1";
    string pattern=@"\d+((?'x'[a-z-[b]]+)).(?<=(?'2-1'(?'x1'..)).{6})b(?(2)(?'Group2Captured'1)|(?'Group2NotCaptured'2))";
    try
    {
    	Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
    	Console.WriteLine($"Interpreted Group2: {matchInterpreted.Groups[2].Captures.Count}");
    	Console.WriteLine($"Interpreted Group2Captured: {matchInterpreted.Groups["Group2Captured"].Captures.Count>0}");
    	Console.WriteLine($"Interpreted Group2NotCaptured: {matchInterpreted.Groups["Group2NotCaptured"].Captures.Count>0}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Interpreted Exception: {ex.Message}");
    }
    
    
    try
    {
    	Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    	Console.WriteLine($"Compiled Group2: {matchCompiled.Groups[2].Captures.Count}");
    	Console.WriteLine($"Compiled Group2Captured: {matchCompiled.Groups["Group2Captured"].Captures.Count>0}");
    	Console.WriteLine($"Compiled Group2NotCaptured: {matchCompiled.Groups["Group2NotCaptured"].Captures.Count>0}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Compiled Exception: {ex.Message}");
    }
Output:

    Interpreted Group2: 0
    Interpreted Group2Captured: True
    Interpreted Group2NotCaptured: False
    Compiled Group2: 0
    Compiled Group2Captured: True
    Compiled Group2NotCaptured: False
In the example above, the capture of Group2 is empty, but when using Group2 as the condition in a conditional expression, it behaves as if it has a capture.  
Using `(?(GroupName)yes|no)` allows you to determine whether a group actually has a capture.   
In this example, if Group2 has no capture, `(?(2)1|2)` should match the `no` branch, which is `2`, but in reality, it matches the `yes` branch, which is `1`.   

However, if the content matched by `exp` is exactly two characters ahead of the latest capture of `g2`, `g2` will behave quite "normally".  
But not entirely normal; after the push and pop operations on Group 2 with `(?'2')(?'-2')`, Group 2 will either have no capture, causing the subsequent `(?(2)1|2)` to match 2, or Group 2 should initially exhibit the property of having a capture.  
    using System.Text.RegularExpressions;
    
    string input = "00123xzacvb21";
    string pattern=@"\d+((?'x'[a-z-[b]]+)).(?<=(?'2-1'(?'x1'..)).{7})b(?(2)(?'Group2Captured'1)|(?'Group2NotCaptured'2))(?'2')(?'-2')(?(2)1|2)";
    try
    {
    	Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
    	Console.WriteLine($"Interpreted Group2: {matchInterpreted.Groups[2].Captures.Count}");
    	Console.WriteLine($"Interpreted Group2Captured: {matchInterpreted.Groups["Group2Captured"].Captures.Count>0}");
    	Console.WriteLine($"Interpreted Group2NotCaptured: {matchInterpreted.Groups["Group2NotCaptured"].Captures.Count>0}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Interpreted Exception: {ex.Message}");
    }
    
    
    try
    {
    	Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    	Console.WriteLine($"Compiled Group2: {matchCompiled.Groups[2].Captures.Count}");
    	Console.WriteLine($"Compiled Group2Captured: {matchCompiled.Groups["Group2Captured"].Captures.Count>0}");
    	Console.WriteLine($"Compiled Group2NotCaptured: {matchCompiled.Groups["Group2NotCaptured"].Captures.Count>0}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Compiled Exception: {ex.Message}");
    }
Output:

    Interpreted Group2: 0
    Interpreted Group2Captured: False
    Interpreted Group2NotCaptured: True
    Compiled Group2: 0
    Compiled Group2Captured: False
    Compiled Group2NotCaptured: True

If a group with negative length capture is back-referenced, it will result in a match failure in interpreter mode. In compiled mode, if the content matched by `exp` is exactly `L` characters ahead of the latest capture of `g2`, it will cause the current input index to move `L` characters in the opposite direction of the match.

    using System.Text.RegularExpressions;
    
    string input = "00123xzacvb1";
    string pattern=@"\d+((?'x'[a-z-[b]]+)).(?<=(?'2-1'(?'x1'.)).{9})b(?(2)(?'Group2Captured'1)|(?'Group2NotCaptured'2))(?'xx'\2)";
    try
    {
    	Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
    	Console.WriteLine($"Interpreted : {matchInterpreted.Success}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Interpreted Exception: {ex.Message}");
    }
    
    
    try
    {
    	Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    	Console.WriteLine($"Compiled : {matchCompiled.Success}");
    	Console.WriteLine($"Compiled Match : {matchCompiled.Index} => {matchCompiled.Index+matchCompiled.Length}");
    	Console.WriteLine($"Compiled Match Value: {matchCompiled.Value}");
    	Console.WriteLine($"Compiled Group2Captured Capture : {matchCompiled.Groups["Group2Captured"].Index} => {matchCompiled.Groups["Group2Captured"].Index+matchCompiled.Groups["Group2Captured"].Length}");
    	Console.WriteLine($"Compiled Group2Captured Capture value : {matchCompiled.Groups["Group2Captured"].Value}");
    	Console.WriteLine($"Compiled Groupxx Capture : {matchCompiled.Groups["xx"].Index} => {matchCompiled.Groups["xx"].Index+matchCompiled.Groups["xx"].Length}");
    	Console.WriteLine($"Compiled Groupxx Capture Value : {matchCompiled.Groups["xx"].Value}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Compiled Exception: {ex.Message}");
    }
Output:

    Interpreted : False
    Compiled : True
    Compiled Match : 0 => 8
    Compiled Match Value: 00123xza
    Compiled Group2Captured Capture : 11 => 12
    Compiled Group2Captured Capture value : 1
    Compiled Groupxx Capture : 8 => 12
    Compiled Groupxx Capture Value : cvb1

For back-referencing a group with negative length capture, although it will move the input index `L` characters in the opposite direction of the match, subsequent matches will still check if the input is long enough at the original index position.

    using System.Text.RegularExpressions;
    
    string input = "xbb3131c";
    string input1 = "xbb3131cz";
    string pattern=@"(?'2'x)(?=.{5}(.))(?'2-1'.)b\d(?(2)1|2)(?(2)3|2)(?'NoMatchWithoutThis'(?'2')(?'-2'))(?'xx'\2)(?'xx1'[a-z])(?'cc'..)";
    try
    {
    	Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input1);
    	Console.WriteLine($"Interpreted : {matchInterpreted.Success}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Interpreted Exception: {ex.Message}");
    }
    
    
    try
    {
    	Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    	Console.WriteLine($"Compiled : {matchCompiled.Success}");
    	
    	matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input1);
    	Console.WriteLine($"Compiled : {matchCompiled.Success}");
    	Console.WriteLine($"Compiled Match : {matchCompiled.Index} => {matchCompiled.Index+matchCompiled.Length}");
    	Console.WriteLine($"Compiled Match Value: {matchCompiled.Value}");
    	Console.WriteLine($"Compiled Group2 Capture : {matchCompiled.Groups[2].Captures.Count}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Compiled Exception: {ex.Message}");
    }
Output:

    Interpreted Exception: Index was outside the bounds of the array.
    Compiled : False
    Compiled : True
    Compiled Match : 0 => 5
    Compiled Match Value: xbb31
    Compiled Group2 Capture : 1
In certain situations, it also seems that the input length is not "checked".

    using System.Text.RegularExpressions;
    
    string input = "000000123xzacb";
    string pattern=@"(?i)\d+((?'x'[a-z-[b]]+)).(?<=(?'2-1'(?'21'..)).{9}[a-z]{0})(?'refer'\2)za";
    try
    {
    	Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
    }catch(Exception ex)
    {
    	Console.WriteLine($"Interpreted Exception: {ex.Message}");
    }
    
    
    try
    {
    	Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    	Console.WriteLine($"Compiled Match: {matchCompiled.Index} => {matchCompiled.Index+matchCompiled.Length}");
    	Console.WriteLine($"Compiled refer: {matchCompiled.Groups["refer"].Index} => {matchCompiled.Groups["refer"].Index+matchCompiled.Groups["refer"].Length}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Compiled Exception: {ex.Message}");
    }
Output:

    Interpreted Exception: Index was outside the bounds of the array.
    Compiled Match: 0 => 12
    Compiled refer: 10 => 14

For a group with negative length capture, performing a balancing group pop operation, if you consider the negative length capture as part of the captures and continue popping until all captures (including negative length captures) are cleared, then the first attempt to retrieve the group information via `Match.Group[groupName]` or `Match.Group[groupNumber]` will throw an exception. On the second attempt, it will return null, and all subsequent groups after this group will also be null.

    using System.Text.RegularExpressions;
    
    string input = "000123xzacvb11";
    string pattern=@"\d+((?'x'[a-z-[b]]+)).(?<=(?'2-1'(?'x1'.)).{9})b(?(2)1|2)(?'2')(?'-2')(?(2)1|2)(?'-2')";
    try
    {
    	Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
    	Console.WriteLine($"Interpreted : {matchInterpreted.Success}");
        var g=matchInterpreted.Groups[2];
    }catch(Exception ex)
    {
    	Console.WriteLine($"Interpreted Exception: {ex.Message}");
    }
    
    
    try
    {
    	Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    	Console.WriteLine($"Compiled : {matchCompiled.Success}");
    	Console.WriteLine($"Compiled Match : {matchCompiled.Index} => {matchCompiled.Index+matchCompiled.Length}");
    	Console.WriteLine($"Compiled Match Value: {matchCompiled.Value}");
    	try
    	{
    		Console.WriteLine($"Compiled Groupx Capture : {matchCompiled.Groups[2].Index} => {matchCompiled.Groups[2].Index+matchCompiled.Groups[2].Length}");
    	}
    	catch(Exception ex1)
    	{
    		Console.WriteLine($"Compiled Exception: {ex1.Message}");
    	}
    	Console.WriteLine($"Compiled Group2 is null: {matchCompiled.Groups[2] is null}");
    	Console.WriteLine($"Compiled Groupx is null: {matchCompiled.Groups["x"] is null}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Compiled Exception: {ex.Message}");
    }
Output:

    Interpreted : True
    Interpreted Exception: Index was outside the bounds of the array.
    Compiled : True
    Compiled Match : 0 => 14
> 
    Compiled Match Value: 000123xzacvb11
    Compiled Exception: Index was outside the bounds of the array.
    Compiled Group2 is null: True
    Compiled Groupx is null: True

Moreover, if the capture group contains both normal content and negative length content, a fake pop phenomenon occurs during the balancing group pop operation.

When the capture group first captures normal content and then captures negative length content, performing the balancing group pop operation will actually only pop the negative length content, leaving the normal content intact. However, querying the captures of the group using `Group.Captures` will show that the capture group has no captures left.

In this situation, using the conditional expression `(?(groupNumber)1|2)` and back-referencing the capture group, you can find that the initially captured normal content is still there.


    using System.Text.RegularExpressions;
    
    string input = "x000123xzacvb11x";
    string pattern=@"(?'2'x)\d+((?'x'[a-z-[b]]+)).(?<=(?'2-1'(?'x1'.)).{9})b(?(2)1|2)(?'-2')(?(2)1|2)(?'group2Capture'\2)";
    try
    {
    	Match matchInterpreted = new Regex(pattern, RegexOptions.None).Match(input);
    	Console.WriteLine($"Interpreted : {matchInterpreted.Success}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Interpreted Exception: {ex.Message}");
    }
    
    
    try
    {
    	Match matchCompiled = new Regex(pattern, RegexOptions.Compiled).Match(input);
    	Console.WriteLine($"Compiled : {matchCompiled.Success}");
    	Console.WriteLine($"Compiled Match : {matchCompiled.Index} => {matchCompiled.Index+matchCompiled.Length}");
    	Console.WriteLine($"Compiled Match Value: {matchCompiled.Value}");
    	Console.WriteLine($"Compiled Group2 Capture : {matchCompiled.Groups[2].Captures.Count}");
    	Console.WriteLine($"Compiled group2Capture Capture : {matchCompiled.Groups["group2Capture"].Index} => {matchCompiled.Groups["group2Capture"].Length}");
    	Console.WriteLine($"Compiled group2Capture Value: {matchCompiled.Groups["group2Capture"].Value}");
    }catch(Exception ex)
    {
    	Console.WriteLine($"Compiled Exception: {ex.Message}");
    }
Output:

    Interpreted : True
    Compiled : True
    Compiled Match : 0 => 16
    Compiled Match Value: x000123xzacvb11x
    Compiled Group2 Capture : 0
    Compiled group2Capture Capture : 15 => 1
    Compiled group2Capture Value: x
