# ``()+?``在解释器下的`特性` #  
``()+?``会使得正则表达式在解释器模式下，即使用`Regex.Match(intut,pattern)` 或 `new Regex(pattern).Match(input)`时，输出结果和在编译模式下，即使用`new Regex(pattern, RegexOptions.Compiled).Matches(input)`时，输出的结果不一致  
而且，在解释器模式下，如果正则表达式包含``()+?``或者等价的子表达式，输出结果会变得非常奇怪。例如：  

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
另一个例子:

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

更多的例子，请参考[GitHub Discussion #110976](https://github.com/dotnet/runtime/discussions/110976)

----------
# 平衡组的bug·其一 #
在某些情况下，当平衡组没有捕获时，应该直接返回匹配失败，不应该错误地抛出异常。  

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
# 平衡组的bug·其二 #
以下的一些例子，会用到比较复杂的正则表达式。
>复杂的正则表达式，可视化可参考[正则可视化与调试工具](https://github.com/longxya/RegexVisual)

在``(?'g1-g2'exp)``，当exp匹配的内容在``g2``的最新捕获内容之前时，``g1``会表现出很奇怪的'特性'  
通过``Group.Captures``查看捕获组的捕获内容，会发现捕获内容为空，但是使用``(?(GroupName)yes|no)``来做判断时却会发现实际上是有捕获内容的，这种情况，暂时称之为``捕获了负长度内容``或``捕获了负长度捕获``  
首先，以代码输出为例:


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
在上面这个例子中，Group2的捕获为空，但是在使用Group2作为条件表达式的判断条件时，却表现出有捕获的性质  
使用``(?(GroupName)yes|no)``可以判断组是否真的有无捕获内容。  
在这个例子中，如果Group2没有捕获，``(?(2)1|2)``应该会匹配``no``分支即``2``，但是实际上匹配的是``yes``即``1``  

但是，如果``exp``匹配的内容恰好比``g2``的最新捕获内容靠前2个字符，``g2``则表现得相当"正常"  
但是又不那么正常，在``(?'2')(?'-2')``对组2进行进栈、出栈后，此时组2要么没有捕获，后续``(?(2)1|2)``会匹配2，要么组2一开始就应该表现出有捕获的性质  

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

如果对这样的有负长度捕获的捕获组进行反向引用，在解释器模式下，会匹配失败，在编译模式下,如果``exp``匹配的内容恰好比``g2``的最新捕获内容靠前L个字符 会使当前匹配的input下标，朝匹配方向相反的方向，移动L个字符长度。

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

对于有负长度捕获的捕获组进行反向引用，虽然会将input的下标向匹配的反方向移动捕获长度个字符，但是在后续匹配，依然会按照原来的下标位置检查input是否够长

    using System.Text.RegularExpressions;
    
    string input = "xbb3131c";
    string input1 = "xbb3131cz";
    string pattern=@"(?'2'x)(?=.{5}(.))(?'2-1'.)b\d(?(2)1|2)(?(2)3|2)(?'如果没有这一步会无匹配'(?'2')(?'-2'))(?'xx'\2)(?'xx1'[a-z])(?'cc'..)";
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
某些情况下，似乎也不会"检查"input是否够长

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


对有负长度捕获的捕获组，进行平衡组出栈，如果把负长度捕获也算进捕获中，一直出栈到所有捕获内容（包括负长度捕获）清空，则第一次通过``Match.Group[groupName]``或``Match.Group[groupNumber]``来获取该捕获组信息时，会抛出异常，第二次获取时会返回null，并且这个组之后的所有组都是null

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
    Compiled Match Value: 000123xzacvb11
    Compiled Exception: Index was outside the bounds of the array.
    Compiled Group2 is null: True
    Compiled Groupx is null: True

而且，如果捕获组包含正常内容，又有负长度内容，则在对该捕获组进行平衡组出栈时，会出现假出栈的现象  
当捕获组先捕获了正常内容，又捕获了负长度内容，对该捕获组进行平衡组出栈时，实际上只会对负长度内容进行出栈，正常内容依然在，但是通过``Group.Captures``查询，会发现捕获组已经没有任何捕获内容了。  
这种情况，通过条件表达式``(?(groupNumber)1|2)``和对捕获组的反向引用，可以发现，第一次捕获的正常内容还在

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
