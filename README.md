# QVTom

QVTom is a prototypical implementation of the module system for model transformation languages described in [[1]](#references). It replaces the default structure of a QVTo transformation with the definition of interfaces and interface implementations where implementations can be linked to interfaces with an export or import relationship.

The changes that have been made to the QVTo plugin [[2]](#references) are described in this document.


## Changes to the AST/CST

### Changes to the meta models
Legend:

Element                       |  Representation
----------------------------- | ----------------
CST	                      |  **CST: ...**
AST	                      |  **AST: ...**
line number in QVTOParser.gi  |  **\<n\>**

<pre>modeltype PCM uses <em>'http://sdq.ipd.uka.de/PalladioComponentModel/5.0'</em>;
modeltype PCM_ALLOC uses <em>'http://sdq.ipd.uka.de/PalladioComponentModel/Allocation/5.0'</em>;
modeltype PCM_REP uses <em>'http://sdq.ipd.uka.de/PalladioComponentModel/Repository/5.0'</em>;

compilation_environment <em>"Commons"</em>;
<b>CST: CompilationEnvironmentCS[uriCS = "Commons"] <547></b>
compilation_environment <em>"EventChannelMiddlewareRegistry"</em>;
compilation_environment <em>"EventDistribution"</em>;
compilation_environment <em>"EventFilter"</em>;
...

interface ISink( 
	inout pcmAllocation : PCM_ALLOC,
    <b>CST: InterfaceInOutParamCS[param = ParameterDeclarationCS[simpleNameCS = "pcmAllocation", typeSpecCS = PCM_ALLOC, directionKind = inout]]</b>
    <b>AST: InterfaceRestrictionParameter[param = VarParameter[parsed by original parser]]</b>
	inout pcmSystem : PCM_SYS, 
	inout pcmRepository : PCM_REP,
	in middlewareRepository : PCM_REP;
	
	in PCM_ALLOC[Allocation]
	  <b>CST: InterfaceRestrictionParamCS[param = PCM_ALLOC, classes = {Allocation::TypeSpecCS}, packages = {}] &lt;780&gt;</b>
	  <b>AST: InterfaceRestrictionParameter</b>
)
  <b>AST: InterfaceParamsCS[…] <680></b>
  <b>AST: ModuleHeaderCS[pathNameCS="ISink", interfaceInOutParamsCS=..., interfaceRestrictionParamsCS=...] &lt;581&gt;</b>
{
	mapping Sink_createSinkOperationProvidedRole(sinkComponent : pcm::repository::RepositoryComponent,
	                                             operationInterface : pcm::repository::OperationInterface) : pcm::repository::OperationProvidedRole;
    <b>&lt;830&gt; - declarations: mapping_decl / helper_decl, Klassen: MappingRuleCS, MappingDeclarationCS (with setBlackbox(true));</b>
}
<b>AST: ModuleInterfaceCS[methods = ..., moduleHeader = ..., metamodels = ModelTypes aus Parametern]</b>
<b>CST: ModuleInterface</b>

 
module Sink mexport ISink
<b>AST: ModuleHeaderCS, ExportCS[pathNameCS = "ISink"] &lt;581&gt;</b>
{
	mimport ISEFFRegistry;  
	mimport ISEFFUtil;
	mimport ICommons;
	mimport IOperationSignatureRegistry;
		
	mapping Sink_createSinkOperationProvidedRole(sinkComponent : pcm::repository::RepositoryComponent,
	                                             operationInterface : pcm::repository::OperationInterface) : pcm::repository::OperationProvidedRole {
			entityName := operationInterface.entityName+<em>'OperationProvidedRole'</em>+Commons_getUniqueElementNameSuffix();
			providingEntity_ProvidedRole := sinkComponent;
			providedInterface__OperationProvidedRole := operationInterface;
	}
	
	<b>&lt;851&gt; - implementations of mapping/helper methods: mapping_def / entry_def / helper_simple_def / helper_compund_def, Klassen: MappingMethodCS, MappingQueryCS, ModulePropertyCS</b>
}
</pre>

### Changes to the grammar QVTOParser.gi
<pre>%Globals
	/.	
	<em>[...]</em>
	import org.eclipse.m2m.internal.qvt.oml.cst.ModuleHeaderCS;
	import org.eclipse.m2m.internal.qvt.oml.cst.MappingMethodCS;
	import org.eclipse.m2m.internal.qvt.oml.cst.InterfaceInOutParamCS;
	import org.eclipse.m2m.internal.qvt.oml.cst.InterfaceParamsCS;
	import org.eclipse.m2m.internal.qvt.oml.cst.InterfaceRestrictionParamCS;
	./
%End

%KeyWords
	<em>[...]</em>
	module
	interface
	mimport
	mexport
	scope
	compilation_environment
%End


%Rules
	<em>[...]</em>
	unit_element -> compilation_env
	
	compilation_env ::= compilation_environment uri ';'
	
	unit_element -> module_def
	unit_element -> interface_def
	
	module_def ::= module_h mexport exportList '{' importList declarationList '}' semicolonOpt
	module_h ::= module qualifiedNameCS

	exportList ::= qualifiedNameCS
	exportList ::= exportList ',' qualifiedNameCS
	
	fullImport ::= mimport qualifiedNameCS ';'
	importList ::= %empty
	importList ::= importList fullImport
	
	interface_def ::= interface_h '{' interfaceList '}' semicolonOpt
	interface_h ::= interface qualifiedNameCS interfaceParams

	interfaceParams ::= '(' interfaceInOutParamList ')'
	interfaceParams ::= '(' interfaceInOutParamList ';' interfaceRestrictionList ')'
	
	interfaceInOutParamList ::= interfaceInOutParamList ',' interfaceInOutParam
	interfaceInOutParamList ::= interfaceInOutParam

	interfaceRestrictionList ::= interfaceRestrictionList ',' interfaceRestrictionParam
	interfaceRestrictionList ::= interfaceRestrictionParam
	interfaceRestrictionParam ::= param_direction typespec '[' typespecList ']'
	interfaceRestrictionParam ::= param_direction typespec '[' typespecList ']' '{' typespecList '}'
		
	paramWithDirection ::= param_direction IDENTIFIER ':' typespec
		
	interfaceInOutParam ::= paramWithDirection
		
	typespecList ::= %empty
	typespecList ::= typespec
	typespecList ::= typespecList ',' typespec
	
	<em>-- returns a MappingRuleCS</em>
	interfaceElement -> mapping_decl
	<em>-- returns a MappingQueryCS</em>
	interfaceElement -> helper_decl
	
	interfaceList ::= interfaceList interfaceElement
	interfaceList ::= interfaceElement

	<em>-- return a MappingMethodCS</em>
	declarationItem -> mapping_def
	declarationItem -> entry_def
	<em>-- return a MappingQueryCS->MappingMethodCS</em>
	declarationItem -> helper_simple_def
	declarationItem -> helper_compound_def
	
	declarationItem -> _property

	declarationList ::= declarationList declarationItem
	declarationList ::= declarationItem
	
	<em>-- changed code ---</em>
	scoped_identifier ::= scoped_identifier2
	scoped_identifier2 ::= IDENTIFIER '@' IDENTIFIER
%End</pre>

### The new metamodels
#### AST
<a href="http://dwerle.github.io/qvtom/images/QVTOperationalAST.png"><img src="http://dwerle.github.io/qvtom/images/QVTOperationalAST_thumb.png" alt="AST metamodel" /></a>
#### CST
<a href="http://dwerle.github.io/qvtom/images/QVTOperationalCST.png"><img src="http://dwerle.github.io/qvtom/images/QVTOperationalCST_thumb.png" alt="CST metamodel" /></a>

### How to include new elements into the CST/AST
0. Once: adapt "/org.eclipse.m2m.qvt.oml.cst.parser/cst/run-lpg.cmd" (LPG_HOME, LPG_EXE, PERL_EXE)
  * Warning: spaces in path names can lead to problems
1. Augment CST and AST metamodels, generate Java code.
  * /org.eclipse.m2m.qvt.oml.cst.parser/model/QVTOperationalCST.ecore
  * /org.eclipse.m2m.qvt.oml/model/QVTOperational.ecore
2. Add new keywords to "/org.eclipse.m2m.qvt.oml.cst.parser/cst/QVTOKWLexer.gi" (export **and** rule)
3. In "/org.eclipse.m2m.qvt.oml.cst.parser/cst/QVTOParser.gi"
  * Import keywords (%KeyWords)
  *	Add rules with reference to factory method in org.eclipse.m2m.internal.qvt.oml.cst.parser.AbstractQVTParser
  * If possible, call `setOffsets` appropriatelyIn der Regel wenn möglich auch setOffsets entsprechend aufrufen.
4. Call "/org.eclipse.m2m.qvt.oml.cst.parser/cst/run-lpg.cmd"
5. If top level elements have been created (such as ModuleImplementation/ModuleInterface/CompilationEnvironment), add them to `org.eclipse.m2m.internal.qvt.oml.cst.parser.AbstractQVTParser.setupTopLevel(EList<CSTNode>)`


### Changes to the CST (/org.eclipse.m2m.qvt.oml.cst.parser/model/QVTOperationalCST.ecore)
* Changes to UnitCS
  * Reference to `ModuleImplementations` and `ModuleInterfaces`
* New: ModuleHeaderCS
* New: ModuleInterfaceCS
* New: InterfaceInOutParamCS
* New: InterfaceRestrictionParamCS
* New: InterfaceParamsCS – eventuell entferenen
* New: ModuleImplementationS
* New: ExportCS
* New: CompilationEnvironmentCS
  * Only references the name of a part of the "Compilation Environment", i.e. all files that have to be compiled together.

### Changes to the AST (/org.eclipse.m2m.qvt.oml/model/QVTOperational.ecore)
* New: ModuleInterface
  * Interface of a module, defines in/out/inout metamodels, visibility of the metamodel and method signatures
* New: InterfaceInOutParameter
  * Same as the in/out/inout parameters of a QVTo transformation but is defined for each ModuleInterface.
* New: InterfaceRestrictionParameter
  * Parameter to restrict the visibility of the metamodel
* New: ModuleImplementation
  * Implementation of a module, exports at least one interface and has to implement at least the methods of the exported interface(s).

### org.eclipse.m2m.internal.qvt.oml.compiler.QVTOCompiler
The entry point for the changes from QVTo to QVTom start in `org.eclipse.m2m.internal.qvt.oml.compiler.QVTOCompiler` in the method `doCompileQVTom`, which replaces `doCompileQVTo`. For the compilation of QVTom in multiple files the following stages are performed:

1. Parse all files that are declared as part of the "Compilation Environment" in the file.
2. Create `ModuleInterfaces` (`createModuleInterfaceStubs`) for every CST, i.e. start the visitor (`QVTOperationalVisitorCS`) for every CST (every file).
3. Resolve all cross references between modules (`crossReferenceModules`). Copy all references to every module interface (the elements that are used to resolve calls to methods of foreign modules) between each two environments of `CSTParseResult`s (of type `QVTOperationalFileEnv`).
4. Compile (CST -> AST) every imported file.
5. Perform a transformation of the resulting QVTom-ASTs to a QVTO-AST which can be interpreted/compiled with the default interpreter/compiler.

### org.eclipse.m2m.internal.qvt.oml.qvtom2qvto.QVTom2QVToCSTransformation
The linking of method calls etc. is already performed during the compilation. For the transformation from QVTom to QVTo the following steps have to be performed:

1. Copy all methods from the module interfaces and module implementations to a new `OperationalTransformation`.
2. Copy in/out/inout parameters to the new transformation.
3. Create properties from the modules in the new transformation.

### Changes to org.eclipse.m2m.internal.qvt.oml.ast.parser.QvtOperationalVisitorCS
Was modified to be able to handle the new syntax elements.

#### lookupModelParameter(SimpleNameCS, DirectionKind, QvtOperationalEnv)
Allows the referencing of `ModelParameter`s that are declared in the interface, i.e. the parsed parameters from the `ModuleInterface` which are `InterfaceInOutParameter`s are made referenceable.

#### QvtOperationalVisitorCS.genOperationCallExp(...)
If no local mapping can be found, check if a fitting method can be found in an imported module. For this purpose traverse upwards in the environment hierarchy until a fitting environment (of type `QvtOperationalFileEnv`) has been found and check if one of the imported interfaces implements a fitting method.

#### visitMappingDeclarationCS(...)
Check the visibility of the context types of the mappings.

#### visitResolveInExpCS(...)
Allow the resolution of methods in the same `ModuleImplementation` or in imported interfaces.

### New methods for modularity in QvtOperationalVisitorCS

#### registerModelTypes(...)
Helper method that is used by `visitModuleInterface`and `visitModuleImplementation`. Parses the used `ModelType`s (`visitModelTypeCS`) and references them in the created Module (`getUsedModelType().add(...)`), adds them as types to the module (`.getEClassifiers().add(...)`) and register them in the environment.

#### visitModuleInterface(...)
Visitor method for `ModuleInterface`s, comparable to the visitor methods for methods.

1. Register `ModelType`s in the environment.
2. Register the `ModelInterface` in the environment.
3. Visit and reference `InOutParameters`and `RestrictionParameter`s.
4. Visit all `MappingDeclaration`s for methods and reference results.

#### visitInOutParamsCS(...)
Visitor methods for `InOutParameter`s.

#### visitRestrictionParamsCS(...)
Visitor method for `RestrictionParam`s. All types are resolved and checked for containment of the package that is to be restricted.

#### visitModuleImplementation(...)
Visitor method for `ModelImplementation`s.

1. Register `ModelType`s in the environment
2. Create `ModelImplementation` and register it in the enviroinment.
3. Create a new module environment ("moduleEnv") which encapsulates the contained methods and represents the module.
   * This is necessary to allow the free distribution of the interfaces and implementations to multiple files.
   * Imported interfaces and variables for used meta models are saved in the interface.
4. Create properties (`createModuleImplementationProperties`)
5. Resolve exports (`resolveExports`), i.e. resolve and reference each interface that is declared as export.
6. Resolve imports, i.e. resolve and reference each interface that is declared as import.
7. Pass through all contained methods
   1. Register contained method in `moduleEnv` (cf. `visitMappingModule`), i.e. create empty methods that can be referenced.
   2. Visit methods (references to empty methods are resolvable).
   3. Check if visibility declarations are violated.
      * `validateMethodMetaModelVisibility(ModuleImplementation, QvtOperationalEnv)`
      * `OCLRestrictedTypeVisitor(ModuleImplementation, QvtOperationalEnv)`
   4. Set entry operation
   5. Export the methods to the interface, cf. `exportMethodsToInterface(...)``
8. Pass the errors from the module environment to the parent environment to make them visible in the IDE.

#### exportMethodsToInterface(...)
In our implementation the method bodies are copied into the interface after the methods are visited (i.e. they are not referenced). We chose this seemingly complicated implementation to allow our modifications to be as non-invasive as possible and to allow the visitor methods for the mappings to perform unchanged.

This method also checks the exported interfaces for complete implementation in the `ModuleImplementation` and reports errors if necessary.

## Meta model visibility validation
The validation of the meta model visibility is realized in two new classes which are explained below. The entry point into the validation can be found in two places:
* `org.eclipse.m2m.internal.qvt.oml.ast.parser.QvtOperationalVisitorCS.validateMethodMetaModelVisibility(ModuleImplementation, QvtOperationalEnv)`
* `QvtOperationalVisitorCS.visitMappingDeclaration(…)`

### OCLRestrictedTypeVisitor
Is created with a `TypeRestrictionSet`or a `ModuleImplementation`. When passing a `ModuleImplementation` the appropriate `.buildFromInterface` method of the `TypeRestrictionSet` is called.

The visitor itself extends `OCLAbstractVisitor`. Every possible violation of the meta model visibility can be checked and annotated with an appropriate error or warning if necessary.

### TypeRestrictionSet
Represents a set of types and packages that are accessible for a particular direction (`DirectionKind`) and possibly for a particular extent.

## QVTo(m)-Environments
Environments are created but not mandatorily persited. They can be used solely for the `QVTOperationalVisitorCS` and discarded afterwards.
* `org.eclipse.m2m.internal.qvt.oml.ast.parser.QvtOperationalVisitorCS.visitObjectExpCS(ObjectExpCS, QvtOperationalEnv, boolean)` -- creates a temporary environment.
* `org.eclipse.m2m.internal.qvt.oml.ast.parser.QvtOperationalVisitorCS.visitModuleImplementation(ModuleImplementationCS, QvtOperationalFileEnv)` -- create a temporary environent for the module so the parsed methods from different modules in the same file context do not interfere.

### org.eclipse.m2m.internal.qvt.oml.ast.env.QvtOperationalEnv
* `org.eclipse.m2m.internal.qvt.oml.ast.env.QvtOperationalEnv.registerMappingOperation(MappingOperation, boolean)`
  * Copy of the equally named method without the boolean parameter. Makes it possible to prevent the registration in the parent environment.
* Additional attributes for the `ModuleImplementation`s and `ModuleInterface`s with respective getters and setters.
* Additional attribute of type `EntryOperation` which represents the entry point for the execution of the transformation.

### org.eclipse.m2m.internal.qvt.oml.ast.env.QvtOperationalModuleEnv, org.eclipse.m2m.internal.qvt.oml.ast.env.QvtOperationalFileEnv
* Concrete subtypes which allow the getting/setting of `ModuleImplementation`s and `ModuleInterface`s.

## Import mechanism of QVTo and QVTom
It is already possible to connect multiple transformations by an import mechanism in QVTo. Details for this mechanism can be found in the QVT standard.

## Workflow of the QVTo(m) parser and interpreter
<img src="http://dwerle.github.io/qvtom/images/Workflow.png" />

Image after <em>Erweitern eines Code-Editors unter Eclipse um neue Sprachkonzepte</em> by Ivayla Partalina.

### Plugins/Packages/Classes
* org.eclipse.m2m.qvt.oml
  * org.eclipse.m2m.internal.qvt.oml.ast.env
    * definiert die benutzten Environments
    * type checking
  * org.eclipse.m2m.internal.qvt.oml.ast.parser
    * QvtOperationalVisitorCS: Konvertierung von CST zu AST
    * OCLRestrictedTypeVisitor / TypeRestrictionSet / ExtentClassifier / ExtentPackage – Überprüfung von Metamodell Sichtbarkeiten (für ECore)
    * OCLAbstractVisitor – Abstrakte Implementierung von org.eclipse.m2m.internal.qvt.oml.expressions.util.QVTOperationalVisitor<Boolean>, als Basis für OCLRestrictedTypeVisitor
    * QvtOperationalParser – Aufsetzen des Root-Elements für das Parseergebnis (UnitCS)
* org.eclipse.m2m.internal.qvt.oml.compiler
  * QVTOCompiler – Einstieg in die Kompilierung. 
* org.eclipse.m2m.internal.qvt.oml.evaluator
  * QvtOperationalEvaluationVisitor / QvtOperationalEvaluationVisitorImpl – Interpreter Schnittstelle + Implementierung, Anpassung an ModulImplementierungen + -Interfaces
* org.eclipse.m2m.internal.qvt.oml.qvtom2qvto
  * QVTom2QVToCSTransformation – Transformation von QVTom nach QVTo (um QVTo-Interpreter zu benutzen)
* org.eclipse.m2m.internal.qvt.oml.cst.parser
  * AbstractQVTParser – Methoden, die aus dem Parser (LPG) aufgerufen werden.
* org.eclipse.m2m.qvt.oml.editor.ui
* org.eclipse.m2m.qvt.oml.runtime
* org.eclipse.m2m.qvt.oml.runtime.ui


## Example for parsing and environments
### Source code
```
modeltype ECORE uses 'http://www.eclipse.org/emf/2002/Ecore';
modeltype UML uses 'http://www.eclipse.org/uml2/4.0.0/UML';

interface I_A(
	in ecore:ECORE,
	out uml:UML
) {
	mapping EClass::EClass2Class() : Class;
}

module A mexport I_A {
	mapping EClass::EClass2Class() : Class {
		name := self.name;
	}
}
```

### Partial dump of the resulting CompiledUnit
```
result = CompiledUnit [
  moduleEnvs = [
    QvtOperationalFileEnv [
      parent = null,
      myFile = ".../minimal.qvto",
      myModuleImplementations = [
        ModuleImplementation[
          name = "A",
          eOperations = [MappingOperation[name="EClass2Class"]]
        ]
      ],
      myModuleInterfaces = [
        ModuleInterface[
          name = "I_A",
          operations = [MappingOperation[name="EClass2Class"]]
        ]
      ]
    ]
  ]
]
```

## See Also
* [Modular Model Transformations](https://sdqweb.ipd.kit.edu/wiki/Modular_Model_Transformations), the overall approach behind this project, as well as information for developers.
* [Xtend2m](http://qvt.github.io/xtend2m/), a modular extension of Xtend hosted at Github.

## References
1. A. Rentschler, D. Werle, Q. Noorshams, L. Happe, R. Reussner. [*Designing Information Hiding Modularity for Model Transformation Languages*](http://dl.acm.org/citation.cfm?doid=2577080.2577094). Proceedings of the 13th International Conference on Modularity (AOSD '14), Lugano, Switzerland, April 2014. ACM, New York, NY, USA. April 2014.
2. [Eclipse Modeling - MMT - Project QVTo](http://www.eclipse.org/mmt/?project=qvto)

## Contributors
* [Dominik Werle](mailto:dominik.werle_AtSignGoesHere_student.kit.edu) from Karlsruhe Institute of Technology
* [Andreas Rentschler] (http://sdq.ipd.kit.edu/people/andreas_rentschler/) from Karlsruhe Institute of Technology
