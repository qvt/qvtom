QVTom is a prototypical implementation of the module system for model transformation languages described in [1]. It replaces the default structure of a QVTo transformation with the definition of interfaces and interface implementations where implementations can be linked to interfaces with an export or import relationship.

The changes that have been made to the QVTo plugin [2] are described in this document.


# Changes to the AST/CST

## Changes to the meta models
Legend:
Element                      | Representation
----------------------------------------------
CST	                         | **CST: ...**
AST	                         | **AST: ...**
line number in QVTOParser.gi | **<n>**

```
modeltype PCM uses 'http://sdq.ipd.uka.de/PalladioComponentModel/5.0';
modeltype PCM_ALLOC uses 'http://sdq.ipd.uka.de/PalladioComponentModel/Allocation/5.0';
modeltype PCM_REP uses 'http://sdq.ipd.uka.de/PalladioComponentModel/Repository/5.0';

compilation_environment "Commons";
  **CST: CompilationEnvironmentCS[uriCS = "Commons"] <547>**
compilation_environment "EventChannelMiddlewareRegistry";
compilation_environment "EventDistribution";
compilation_environment "EventFilter";
...

interface ISink( 
	inout pcmAllocation : PCM_ALLOC,
    **CST: InterfaceInOutParamCS[param = ParameterDeclarationCS[simpleNameCS = "pcmAllocation", typeSpecCS = PCM_ALLOC, directionKind = inout]]**
    **AST: InterfaceRestrictionParameter[param = VarParameter[parsed by original parser]]**
	inout pcmSystem : PCM_SYS, 
	inout pcmRepository : PCM_REP,
	in middlewareRepository : PCM_REP;
	
	in PCM_ALLOC[Allocation]
	  **CST: InterfaceRestrictionParamCS[param = PCM_ALLOC, classes = {Allocation::TypeSpecCS}, packages = {}] <780>**
	  **AST: InterfaceRestrictionParameter**
)
  **AST: InterfaceParamsCS[…] <680>**
  **AST: ModuleHeaderCS[pathNameCS="ISink", interfaceInOutParamsCS=..., interfaceRestrictionParamsCS=...] <581>**
{
	mapping Sink_createSinkOperationProvidedRole(sinkComponent : pcm::repository::RepositoryComponent,
	                                             operationInterface : pcm::repository::OperationInterface) : pcm::repository::OperationProvidedRole;
    **<830> - declarations: mapping_decl / helper_decl, Klassen: MappingRuleCS, MappingDeclarationCS (with setBlackbox(true));**
}
**AST: ModuleInterfaceCS[methods = ..., moduleHeader = ..., metamodels = ModelTypes aus Parametern]**
**CST: ModuleInterface**

 
module Sink mexport ISink
  **AST: ModuleHeaderCS, ExportCS[pathNameCS = "ISink"] <581>**
{
	mimport ISEFFRegistry;  
	mimport ISEFFUtil;
	mimport ICommons;
	mimport IOperationSignatureRegistry;
		
	mapping Sink_createSinkOperationProvidedRole(sinkComponent : pcm::repository::RepositoryComponent,
	                                             operationInterface : pcm::repository::OperationInterface) : pcm::repository::OperationProvidedRole {
			entityName := operationInterface.entityName+'OperationProvidedRole'+Commons_getUniqueElementNameSuffix();
			providingEntity_ProvidedRole := sinkComponent;
			providedInterface__OperationProvidedRole := operationInterface;
	}
	
	**<851> - implementations of mapping/helper methods: mapping_def / entry_def / helper_simple_def / helper_compund_def, Klassen: MappingMethodCS, MappingQueryCS, ModulePropertyCS**
}

## Changes to the grammar QVTOParser.gi
```
%Globals
	/.	
	[...]
	import org.eclipse.m2m.internal.qvt.oml.cst.ModuleHeaderCS;
	import org.eclipse.m2m.internal.qvt.oml.cst.MappingMethodCS;
	import org.eclipse.m2m.internal.qvt.oml.cst.InterfaceInOutParamCS;
	import org.eclipse.m2m.internal.qvt.oml.cst.InterfaceParamsCS;
	import org.eclipse.m2m.internal.qvt.oml.cst.InterfaceRestrictionParamCS;
	./
%End

%KeyWords
	[...]
	module
	interface
	mimport
	mexport
	scope
	compilation_environment
%End


%Rules
	[...]
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
	
	-- returns a MappingRuleCS
	interfaceElement -> mapping_decl
	-- returns a MappingQueryCS
	interfaceElement -> helper_decl
	
	interfaceList ::= interfaceList interfaceElement
	interfaceList ::= interfaceElement

	-- return a MappingMethodCS
	declarationItem -> mapping_def
	declarationItem -> entry_def
	-- return a MappingQueryCS->MappingMethodCS
	declarationItem -> helper_simple_def
	declarationItem -> helper_compound_def
	
	declarationItem -> _property

	declarationList ::= declarationList declarationItem
	declarationList ::= declarationItem
	
	-- changed code ---
	scoped_identifier ::= scoped_identifier2
	scoped_identifier2 ::= IDENTIFIER '@' IDENTIFIER
%End
```

## The new metamodels
### AST
<img src="http://qvt.github.io/qvtom/images/metamodelAST.png" alt="AST metamodel" />
### CST
<img src="http://qvt.github.io/qvtom/images/metamodelCST.png" alt="CST metamodel" />

## How to include new elements into the CST/AST
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


## Changes to the CST (/org.eclipse.m2m.qvt.oml.cst.parser/model/QVTOperationalCST.ecore)
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

## Changes to the AST (/org.eclipse.m2m.qvt.oml/model/QVTOperational.ecore)
* New: ModuleInterface
  * Interface of a module, defines in/out/inout metamodels, visibility of the metamodel and method signatures
* New: InterfaceInOutParameter
  * Same as the in/out/inout parameters of a QVTo transformation but is defined for each ModuleInterface.
* New: InterfaceRestrictionParameter
  * Parameter to restrict the visibility of the metamodel
* New: ModuleImplementation
  * Implementation of a module, exports at least one interface and has to implement at least the methods of the exported interface(s).
 	
# Example for parsing and environments
## Source code
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

## Partial dump of the resulting CompiledUnit
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

# See Also
* [Modular Model Transformations](https://sdqweb.ipd.kit.edu/wiki/Modular_Model_Transformations), the overall approach behind this project, as well as information for developers.
* [Xtend2m](http://qvt.github.io/xtend2m/), a modular extension of Xtend hosted at Github.

# References
* [1] A. Rentschler, D. Werle, Q. Noorshams, L. Happe, R. Reussner. [*Designing Information Hiding Modularity for Model Transformation Languages*](http://dl.acm.org/citation.cfm?doid=2577080.2577094). Proceedings of the 13th International Conference on Modularity (AOSD '14), Lugano, Switzerland, April 2014. ACM, New York, NY, USA. April 2014.
* [2] [Eclipse Modeling - MMT - Project QVTo](http://www.eclipse.org/mmt/?project=qvto)

# Contributors
* [Dominik Werle](emailto:dominik.werle_AtSignGoesHere_student.kit.edu) from Karlsruhe Institute of Technology
* [Andreas Rentschler] (http://sdq.ipd.kit.edu/people/andreas_rentschler/) from Karlsruhe Institute of Technology
