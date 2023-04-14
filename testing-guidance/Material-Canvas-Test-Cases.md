# Material-Canvas-Test-Cases


## Test 1
Re-create "test1.materialgraph" and compare shader file output to original

- Build the O3de AutomatedTesting project 
 - Have access to Beyond Compare or other file comparison tool


1. Launch MaterialCanvas.exe
2. Go to: File > Open
3. Open the test1.materialgraph from "\o3de\Gems\Atom\Tools\MaterialCanvas\Assets\MaterialCanvas\TestData"
4. Copy all test1 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test1_original"
5. Go to: File > New > New Material Graph Document...
6. Create a new "test1" document, at the default location "\o3de\AutomatedTesting\Assets"
7. Add "Float4 Constant" to the canvas, set the X value to 1, and the Z value to one
8. Add "Bool Input" and "Base PBR" to the canvas
9. Connect the "Float4 Constant" value output to the "Base PBR" Metallic input
10. Copy all test1 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test1_new"
11. Compare the contents of "test1_original" and "test1_new"

- "test1_original" and "test1_new" match

- The node identifiers (such as "node2" or "node10") differ between the the newly created graph document and the original. Will need to find a way to make them match, or exclude them from the compare
 - See attachment for screenshot of graph


## Test 2
Re-create "test2.materialgraph" and compare shader file output to original

- Build the O3de AutomatedTesting project 
 - Have access to Beyond Compare or other file comparison tool

1. Launch MaterialCanvas.exe
2. Go to: File > Open
3. Open the test2.materialgraph from "\o3de\Gems\Atom\Tools\MaterialCanvas\Assets\MaterialCanvas\TestData"
4. Copy all test2 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test2_original"
5. Go to: File > New > New Material Graph Document...
6. Create a new "test2" document, at the default location "\o3de\AutomatedTesting\Assets"
7. Add a "Float4 Constant" node to the canvas, and set the X,Z, and W value to 1
8. Add "Time", "Sine", and "Absolute Value" nodes to your graph
9. Connect Time output to Sine input, and Sine output to Absolute Value input
10. Add a "Multiply" node to the graph
11. Connect the "Float4Constant" Value output to the "Multiply" Value1 input
12. Connect the "Absolute Value" Value output to the "Multiply" Value2 input
13. Add a 2nd "Float4 Constant" node, and set the X and W values to 1
14. Add a 2nd "Multiply" node to the canvas
15. Connect the 2nd "Float4 Constant" node output value, to the 2nd "Multiply" Value1 input
16. Connect the 1st "Multiply" node output value, to the 2nd "Multiply" node's Value2 input
17. Add the "Standard PBR" to the graph
18. Connect 2nd "Multiply" output value to the "Standard PBR" Base Color value
19. Connect 1st "Multiply" output value to the "Standard PBR" Emissive value
20. Copy all test2 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test2_new"
21. Compare the contents of "test2_original" and "test2_new"


- "test2_original" and "test2_new" match

- The node identifiers (such as "node2" or "node10") differ between the the newly created graph document and the original. Will need to find a way to make them match, or exclude them from the compare
 - See attachment for screenshot of graph


## Test 3
Re-create "test3.materialgraph" and compare shader file output to original

- Build the O3de AutomatedTesting project 
 - Have access to Beyond Compare or other file comparison tool

1. Launch MaterialCanvas.exe
2. Go to: File > Open
3. Open the test3.materialgraph from "\o3de\Gems\Atom\Tools\MaterialCanvas\Assets\MaterialCanvas\TestData"
4. Copy all test3 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test3_original"
5. Go to: File > New > New Material Graph Document...
6. Create a new "test3" document, at the default location "\o3de\AutomatedTesting\Assets"
7. Add a "UV" node to the graph, set its Index value to 1
8. Add a "Sample Texture 2d" to the graph, click on the node, and in the inspector panel select "checker8x8_flip_512" from the Image value
9. Add "Base PBR" node to the graph
10. Connect the "UV" nodes UV output to the "Sample Texture 2d" UV input
11. Connect the "Sample Texture 2d" color output, to the "Base PBR" Base Color input
12. Copy all test3 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test3_new"
13. Compare the contents of "test3_original" and "test3_new"

- "test3_original" and "test3_new" match

- The node identifiers (such as "node2" or "node10") differ between the the newly created graph document and the original. Will need to find a way to make them match, or exclude them from the compare
 - See attachment for screenshot of graph


## Test 4
Re-create "test4.materialgraph" and compare shader file output to original

- Build the O3de AutomatedTesting project 
 - Have access to Beyond Compare or other file comparison tool

1. Launch MaterialCanvas.exe
2. Go to: File > Open
3. Open the test4.materialgraph from "\o3de\Gems\Atom\Tools\MaterialCanvas\Assets\MaterialCanvas\TestData"
4. Copy all test4 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test4_original"
5. Go to: File > New > New Material Graph Document...
6. Create a new "test4" document, at the default location "\o3de\AutomatedTesting\Assets"
7. Add a "UV" node to the graph, set its Index value to 1
8. Add a "Sample Texture 2d" to the graph, click on the node, and in the inspector panel select "checker8x8_flip_512" from the Image value
9. Add "Base PBR" node to the graph
10. Connect the "UV" nodes UV output to the "Sample Texture 2d" UV input
11. Connect the "Sample Texture 2d" color output, to the "Base PBR" Base Color input
12. Add a "World Position" node to the graph, and connect its Z output to the "Base PBR" Metallic input
12. Copy all test4 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test4_new"
13. Compare the contents of "test4_original" and "test4_new"

- "test4_original" and "test4_new" match

- The node identifiers (such as "node2" or "node10") differ between the the newly created graph document and the original. Will need to find a way to make them match, or exclude them from the compare
 - See attachment for screenshot of graph


## Test 5
Re-create "test5.materialgraph" and compare shader file output to original

- Build the O3de AutomatedTesting project 
 - Have access to Beyond Compare or other file comparison tool

1. Launch MaterialCanvas.exe
2. Go to: File > Open
3. Open the test5.materialgraph from "\o3de\Gems\Atom\Tools\MaterialCanvas\Assets\MaterialCanvas\TestData"
4. Copy all test5 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test5_original"
5. Go to: File > New > New Material Graph Document...
6. Create a new "test5" document, at the default location "\o3de\AutomatedTesting\Assets"
7. Add a "UV", and "Sample Texture 2d", and a "Standard PBR" 
8. Set the UV node's Index value to 1, connect the UV output to the "Sample Texture 2d" node's UV input
9. Click on the "Sample Texture 2d" node and go to the inspector panel, set the Image property to "checker8x8_512"
10. Connect the "Sample Texture 2d" node's Color output to the "Standard PBR" node's Emissive input
11. Add a "Float4Constant" node to the graph, set its the X value to 1, and W value to 10
12. Add a "Linear Interpolate" node to the graph
13. Connect the "Float4Constant" node's Value output to the "Linear Interpolate" node's Value1 input
14. Add a 2nd "Float4Constant" node to the graph, set its the X value to 0.1, and W value to 1
15. Connect the 2nd "Float4Constant" node's Value output to the "Linear Interpolate" node's Value2 input
16. Add a "Time", "Cosine". and "Sine" node to the graph
17. Connect the "Time" output to the "Cosine" and "Sine" Value input
18. Add a "Multiply" and "Ceiling" node to the graph
19. Connect the "Cosine" Value output to the "Multiply" node's Value1, and Value2 inputs
20. Connect the "Sine" Value output to the "Ceiling" Value input
21. Add a "Absolute Value" node to the graph
22. Connect the "Sine" Value output to the "Absolute Value" Value input
23. Connect the "Absolute Value" Value output to the "Linear Interpolate" node's T input
24. Connect the "Multiply" node's Value output, to the "Standard PBR" node's Metallic input
25. Add a 2nd "Multiply" node to the graph
26. Connect the 1st "Multiply" node's Value output, to the 2nd "Multiply" node's Value2 input
27. Add a "Normal" node to the graph
28. Connect the "Normal" node's Normal output to the 2nd "Multiply" node's Value1 input
29. Connect the 2nd "Multiply" node's value output to the "Standard PBR" node's Position Offset input
30. Connect the "Linear Interpolate" node's Value output to "Standard PBR" node's Base Color input
31. Connect the "Ceiling" node's Value output to the "Standard PBR" node's Roughness input
32. Copy all test5 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test5_new"
33. Compare the contents of "test5_original" and "test5_new"

- "test5_original" and "test5_new" match

- The node identifiers (such as "node2" or "node10") differ between the the newly created graph document and the original. Will need to find a way to make them match, or exclude them from the compare
 - See attachment for screenshot of graph

## Test 6
Re-create "test8.materialgraph" and compare shader file output to original

- Build the O3de AutomatedTesting project 
 - Have access to Beyond Compare or other file comparison tool

1. Launch MaterialCanvas.exe
2. Go to: File > Open
3. Open the test8.materialgraph from "\o3de\Gems\Atom\Tools\MaterialCanvas\Assets\MaterialCanvas\TestData"
4. Copy all test8 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test8_original"
5. Go to: File > New > New Material Graph Document...
6. Create a new "test8" document, at the default location "\o3de\AutomatedTesting\Assets"
7. Add a "Float Input" node to the graph, set it's name property to "Speed", and the Value property to 10
8. Add "Time" and "Multiply" nodes to the graph
9. Connect "Float Input" output Value to "Multiply" input Value1
10. Connect the "Time" node's Time value output to "Multiply" input Value2
11. Add a "Cosine" and "Normal" node
12. Connect the "Multiply" Value output to the "Cosine" Value input
13. Add a 2nd "Multiply" node to the graph
14. Connect the "Normal" node's Normal output to the 2nd "Multiply" node's Value1
15. Connect the "Cosine" node's Value output to the 2nd "Multiply" node's Value2
16. Add an "Absolute Value" node to the graph
17. Connect the 2nd "Multiply" node's Value output to the "Absolute Value" node's Value output
18. Add a "Base PBR" node to the graph
19. Connect the "Absolute Value" node's Value output to the "Base PBR" node's Base Color input
20. Copy all test8 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test8_new"
21. Compare the contents of "test8_original" and "test8_new"

"test8_original" and "test8_new" match

- The node identifiers (such as "node2" or "node10") differ between the the newly created graph document and the original. Will need to find a way to make them match, or exclude them from the compare
 - See attachment for screenshot of graph


## Test 7
Re-create "test9.materialgraph" and compare shader file output to original

- Build the O3de AutomatedTesting project 
 - Have access to Beyond Compare or other file comparison tool

1. Launch MaterialCanvas.exe
2. Go to: File > Open
3. Open the test9.materialgraph from "\o3de\Gems\Atom\Tools\MaterialCanvas\Assets\MaterialCanvas\TestData"
4. Copy all test9 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test9_original"
5. Go to: File > New > New Material Graph Document...
6. Create a new "test9" document, at the default location "\o3de\AutomatedTesting\Assets"
7. Add a "UV, "Sample Texture 2d", and "Standard PBR"
8. Select the "UV" node, and set the Index value to 1
9. Select the "Sample Texture 2d", go the Inspector, and set the Image to file "checker8x8_gray_512.png"
10. Connect the "UV" node's UV output to the "Sample Texture 2d" node's UV input
11. Connect the "Sample Texture 2d" node's R output to the "Standard PBR" node's Alpha value
12. Copy all test9 files generated (*.material, *.materialtype, *.shader, *.azsl, and *.azsli) to a folder called "test9_new"
13. Compare the contents of "test9_original" and "test9_new"

- "test9_original" and "test9_new" match

- The node identifiers (such as "node2" or "node10") differ between the the newly created graph document and the original. Will need to find a way to make them match, or exclude them from the compare
 - See attachment for screenshot of graph