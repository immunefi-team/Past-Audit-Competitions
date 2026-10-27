# #37359 \[BC-Insight] Failure to Generate ABI Binding in Golang

**Submitted on Dec 2nd 2024 at 20:36:56 UTC by @CertiK for** [**Attackathon | Ethereum Protocol**](https://immunefi.com/audit-competition/ethereum-protocol-attackathon)

* **Report ID:** #37359
* **Report Type:** Blockchain/DLT
* **Report severity:** Insight
* **Target:** <https://github.com/ledgerwatch/erigon>
* **Impacts:**
  * (Specifications) A bug in specifications with no direct impact on client implementations

## Description

## Brief/Intro

Abigen fails to generate ABI binding of Solidity code in the accounts/abi package when the keyword of Golang is passed as the inputs of the binding.

## Vulnerability Details

Affected Codebase:\
<https://github.com/erigontech/erigon/tree/v2.61.0-beta1>

The Ethereum client Erigon (<https://github.com/erigontech/erigon>) provides the Abigen tool to generate the Golang wrapper around a Solidity contract ABI:

<https://github.com/erigontech/erigon/blob/v2.61.0-beta1/accounts/abi/bind/bind.go#L60>

```
func Bind(types []string, abis []string, bytecodes []string, fsigs []map[string]string, pkg string, lang Lang, libs map[string]string, aliases map[string]string) (string, error) {
	var (
		// contracts is the map of each individual contract requested binding
		contracts = make(map[string]*tmplContract)


		// structs is the map of all redeclared structs shared by passed contracts.
		structs = make(map[string]*tmplStruct)


		// isLib is the map used to flag each encountered library as such
		isLib = make(map[string]struct{})
	)
	for i := 0; i < len(types); i++ {
		// Parse the actual ABI to generate the binding for
		evmABI, err := abi.JSON(strings.NewReader(abis[i]))
		if err != nil {
			return "", err
		}
		// Strip any whitespace from the JSON ABI
		strippedABI := strings.Map(func(r rune) rune {
			if unicode.IsSpace(r) {
				return -1
			}
			return r
		}, abis[i])


		// Extract the call and transact methods; events, struct definitions; and sort them alphabetically
		var (
			calls     = make(map[string]*tmplMethod)
			transacts = make(map[string]*tmplMethod)
			events    = make(map[string]*tmplEvent)
			fallback  *tmplMethod
			receive   *tmplMethod


			// identifiers are used to detect duplicated identifiers of functions
			// and events. For all calls, transacts and events, abigen will generate
			// corresponding bindings. However we have to ensure there is no
			// identifier collisions in the bindings of these categories.
			callIdentifiers     = make(map[string]bool)
			transactIdentifiers = make(map[string]bool)
			eventIdentifiers    = make(map[string]bool)
		)
		for _, original := range evmABI.Methods {
			// Normalize the method for capital cases and non-anonymous inputs/outputs
			normalized := original
			normalizedName := methodNormalizer[lang](alias(aliases, original.Name))
			// Ensure there is no duplicated identifier
			var identifiers = callIdentifiers
			if !original.IsConstant() {
				identifiers = transactIdentifiers
			}
			if identifiers[normalizedName] {
				return "", fmt.Errorf("duplicated identifier \"%s\"(normalized \"%s\"), use --alias for renaming", original.Name, normalizedName)
			}
			identifiers[normalizedName] = true
			normalized.Name = normalizedName
			normalized.Inputs = make([]abi.Argument, len(original.Inputs))
			copy(normalized.Inputs, original.Inputs)
			for j, input := range normalized.Inputs {
				if input.Name == "" {
					normalized.Inputs[j].Name = fmt.Sprintf("arg%d", j)
				}
				if hasStruct(input.Type) {
					bindStructType[lang](input.Type, structs)
				}
			}
			normalized.Outputs = make([]abi.Argument, len(original.Outputs))
			copy(normalized.Outputs, original.Outputs)
			for j, output := range normalized.Outputs {
				if output.Name != "" {
					normalized.Outputs[j].Name = capitalise(output.Name)
				}
				if hasStruct(output.Type) {
					bindStructType[lang](output.Type, structs)
				}
			}
			// Append the methods to the call or transact lists
			if original.IsConstant() {
				calls[original.Name] = &tmplMethod{Original: original, Normalized: normalized, Structured: structured(original.Outputs)}
			} else {
				transacts[original.Name] = &tmplMethod{Original: original, Normalized: normalized, Structured: structured(original.Outputs)}
			}
		}
		for _, original := range evmABI.Events {
			// Skip anonymous events as they don't support explicit filtering
			if original.Anonymous {
				continue
			}
			// Normalize the event for capital cases and non-anonymous outputs
			normalized := original


			// Ensure there is no duplicated identifier
			normalizedName := methodNormalizer[lang](alias(aliases, original.Name))
			if eventIdentifiers[normalizedName] {
				return "", fmt.Errorf("duplicated identifier \"%s\"(normalized \"%s\"), use --alias for renaming", original.Name, normalizedName)
			}
			eventIdentifiers[normalizedName] = true
			normalized.Name = normalizedName


			normalized.Inputs = make([]abi.Argument, len(original.Inputs))
			copy(normalized.Inputs, original.Inputs)
			for j, input := range normalized.Inputs {
				if input.Name == "" {
					normalized.Inputs[j].Name = fmt.Sprintf("arg%d", j)
				}
				if hasStruct(input.Type) {
					bindStructType[lang](input.Type, structs)
				}
			}
			// Append the event to the accumulator list
			events[original.Name] = &tmplEvent{Original: original, Normalized: normalized}
		}
		// Add two special fallback functions if they exist
		if evmABI.HasFallback() {
			fallback = &tmplMethod{Original: evmABI.Fallback}
		}
		if evmABI.HasReceive() {
			receive = &tmplMethod{Original: evmABI.Receive}
		}
		// There is no easy way to pass arbitrary java objects to the Go side.
		if len(structs) > 0 && lang == LangJava {
			return "", errors.New("java binding for tuple arguments is not supported yet")
		}


		contracts[types[i]] = &tmplContract{
			Type:        capitalise(types[i]),
			InputABI:    strings.ReplaceAll(strippedABI, "\"", "\\\""),
			InputBin:    strings.TrimPrefix(strings.TrimSpace(bytecodes[i]), "0x"),
			Constructor: evmABI.Constructor,
			Calls:       calls,
			Transacts:   transacts,
			Fallback:    fallback,
			Receive:     receive,
			Events:      events,
			Libraries:   make(map[string]string),
		}
		// Function 4-byte signatures are stored in the same sequence
		// as types, if available.
		if len(fsigs) > i {
			contracts[types[i]].FuncSigs = fsigs[i]
		}
		// Parse library references.
		for pattern, name := range libs {
			matched, err := regexp.Match("__\\$"+pattern+"\\$__", []byte(contracts[types[i]].InputBin))
			if err != nil {
				log.Error("Could not search for pattern", "pattern", pattern, "contract", contracts[types[i]], "err", err)
			}
			if matched {
				contracts[types[i]].Libraries[pattern] = name
				// keep track that this type is a library
				if _, ok := isLib[name]; !ok {
					isLib[name] = struct{}{}
				}
			}
		}
	}
	// Check if that type has already been identified as a library
	for i := 0; i < len(types); i++ {
		_, ok := isLib[types[i]]
		contracts[types[i]].Library = ok
	}
	// Generate the contract template data content and render it
	data := &tmplData{
		Package:   pkg,
		Contracts: contracts,
		Libraries: libs,
		Structs:   structs,
	}
	buffer := new(bytes.Buffer)


	funcs := map[string]interface{}{
		"bindtype":      bindType[lang],
		"bindtopictype": bindTopicType[lang],
		"namedtype":     namedType[lang],
		"capitalise":    capitalise,
		"decapitalise":  decapitalise,
	}
	tmpl := template.Must(template.New("").Funcs(funcs).Parse(tmplSource[lang]))
	if err := tmpl.Execute(buffer, data); err != nil {
		return "", err
	}
	// For Go bindings pass the code through gofmt to clean it up
	if lang == LangGo {
		code, err := format.Source(buffer.Bytes())
		if err != nil {
			return "", fmt.Errorf("%w\n%s", err, buffer)
		}
		return string(code), nil
	}
	// For all others just return as is for now
	return buffer.String(), nil
}
```

However, in case that the Golang keyword is passed in as the function parameter names, the ABI wrapper generation fails.

This issue of Abigen has been reported in the Go-ethereum: <https://github.com/ethereum/go-ethereum/issues/25252>

And it has been patched in the PR:\
<https://github.com/ethereum/go-ethereum/pull/25307>

## Impact Details

The tool Abigen fails in case that the Golang keywords are passed as the function parameter names.

## References

* <https://github.com/erigontech/erigon/tree/v2.61.0-beta1>
* <https://github.com/ethereum/go-ethereum/issues/25252>
* <https://github.com/ethereum/go-ethereum/pull/25307>

## Proof of Concept

## Proof of Concept

For simplicity, we can reuse and modify the test case from go-ethereum (<https://github.com/ethereum/go-ethereum/pull/25307> ) to verify the issue:

```
 // Test Golang Keyword conflict, for example, range keword
   {
      `RangeKeyword`,
      `
      // SPDX-License-Identifier: GPL-3.0
      pragma solidity >=0.4.22 <0.9.0;
      contract keywordcontract {
         function functionWithKeywordParameter(range uint256) public pure {}
      }
      `,
      []string{"0x608060405234801561001057600080fd5b5060dc8061001f6000396000f3fe6080604052348015600f57600080fd5b506004361060285760003560e01c8063527a119f14602d575b600080fd5b60436004803603810190603f9190605b565b6045565b005b50565b6000813590506055816092565b92915050565b600060208284031215606e57606d608d565b5b6000607a848285016048565b91505092915050565b6000819050919050565b600080fd5b6099816083565b811460a357600080fd5b5056fea2646970667358221220d4f4525e2615516394055d369fb17df41c359e5e962734f27fd683ea81fd9db164736f6c63430008070033"},
      []string{`[{"inputs":[{"internalType":"uint256","name":"range","type":"uint256"}],"name":"functionWithKeywordParameter","outputs":[],"stateMutability":"pure","type":"function"}]`},
      `
         "context"
         "math/big"


         "github.com/ledgerwatch/erigon/accounts/abi/bind"
         "github.com/ledgerwatch/erigon/accounts/abi/bind/backends"
         "github.com/ledgerwatch/erigon/types"
         "github.com/ledgerwatch/erigon/crypto"
         "github.com/ledgerwatch/erigon/eth/ethconfig"
      `,
      `
var (
            key, _  = crypto.GenerateKey()
            user, _ = bind.NewKeyedTransactorWithChainID(key, big.NewInt(1337))
            sim     = backends.NewSimulatedBackend(core.GenesisAlloc{user.From: {Balance: big.NewInt(1000000000000000000)}}, ethconfig.Defaults.Miner.GasCeil)
         )
         _, tx, _, err := DeployRangeKeyword(user, sim)
         if err != nil {
            t.Fatalf("error deploying contract: %v", err)
         }
         sim.Commit()
         if _, err = bind.WaitDeployed(nil, sim, tx); err != nil {
            t.Errorf("error deploying the contract: %v", err)
         }
   `,
      nil,
      nil,
      nil,
      nil,
   }
```

Run the following unit test:

```
// Tests that packages generated by the binder can be successfully compiled and
// the requested tester run against it.
func TestGolangBindings(t *testing.T) {
   // Skip the test if no Go command can be found
   //gocmd := "go"
   gocmd := runtime.GOROOT() + "/bin/go"
   if !dir.FileExist(gocmd) {
      t.Skip("go sdk not found for testing")
   }
   // Create a temporary workspace for the test suite
   ws := t.TempDir()


   pkg := filepath.Join(ws, "bindtest")
   if err := os.MkdirAll(pkg, 0700); err != nil {
      t.Fatalf("failed to create package: %v", err)
   }
   // Generate the test suite for all the contracts
   for i, tt := range bindTests {
      var types []string
      if tt.types != nil {
         types = tt.types
      } else {
         types = []string{tt.name}
      }
      // Generate the binding and create a Go source file in the workspace
      bind, err := Bind(types, tt.abi, tt.bytecode, tt.fsigs, "bindtest", LangGo, tt.libs, tt.aliases)
      if err != nil {
         t.Fatalf("test %d: failed to generate binding: %v", i, err)
      }
      if err = os.WriteFile(filepath.Join(pkg, strings.ToLower(tt.name)+".go"), []byte(bind), 0600); err != nil {
         t.Fatalf("test %d: failed to write binding: %v", i, err)
      }
      // Generate the test file with the injected test code
      code := fmt.Sprintf(`
         package bindtest


         import (
            "testing"
            %s
         )


         func Test%s(t *testing.T) {
            %s
         }
      `, tt.imports, tt.name, tt.tester)
      if err := os.WriteFile(filepath.Join(pkg, strings.ToLower(tt.name)+"_test.go"), []byte(code), 0600); err != nil {
         t.Fatalf("test %d: failed to write tests: %v", i, err)
      }
   }
   // Convert the package to go modules and use the current source for go-ethereum
   moder := exec.Command(gocmd, "mod", "init", "bindtest")
   moder.Dir = pkg
   if out, err := moder.CombinedOutput(); err != nil {
      t.Fatalf("failed to convert binding test to modules: %v\n%s", err, out)
   }
   pwd, _ := os.Getwd()
   replacer := exec.Command(gocmd, "mod", "edit", "-replace", "github.com/ledgerwatch/erigon="+filepath.Join(pwd, "..", "..", "..")) // Repo root
   replacer.Dir = pkg
   if out, err := replacer.CombinedOutput(); err != nil {
      t.Fatalf("failed to replace binding test dependency to current source tree: %v\n%s", err, out)
   }


   tidier := exec.Command(gocmd, "mod", "tidy")
   tidier.Dir = pkg
   if out, err := tidier.CombinedOutput(); err != nil {
      t.Fatalf("failed to tidy Go module file: %v\n%s", err, out)
   }
   //Test the entire package and report any failures
   cmd := exec.Command(gocmd, "test", "-v", "-count", "1")
   cmd.Dir = pkg
   if out, err := cmd.CombinedOutput(); err != nil {
      t.Fatalf("failed to run binding test: %v\n%s", err, out)
   }
}
```

The test result shows the Abigen fails due to the `range` keyword conflict:

```
=== RUN   TestGolangBindings
    bind_test.go:1859: test 28: failed to generate binding: 208:95: expected ')', found 'range' (and 9 more errors)
        
        // Code generated by abigen. DO NOT EDIT.
        // This file is a generated binding and any manual changes will be lost.
        
        package bindtest
        
        import (
        	"math/big"
        	"strings"
        	"fmt"
        	"reflect"
        
        	ethereum "github.com/ledgerwatch/erigon"
        	"github.com/ledgerwatch/erigon/accounts/abi"
        	"github.com/ledgerwatch/erigon/accounts/abi/bind"
        	libcommon "github.com/ledgerwatch/erigon-lib/common"
        	"github.com/ledgerwatch/erigon/core/types"
        	"github.com/ledgerwatch/erigon/event"
        )
        
        // Reference imports to suppress errors if they are not otherwise used.
        var (
        	_ = big.NewInt
        	_ = strings.NewReader
        	_ = ethereum.NotFound
        	_ = bind.Bind
        	_ = libcommon.Big1
        	_ = types.BloomLookup
        	_ = event.NewSubscription
        	_ = fmt.Errorf
        	_ = reflect.ValueOf
        )
        
        
        
        
        
        	// RangeKeywordABI is the input ABI used to generate the binding from.
        	const RangeKeywordABI = "[{\"inputs\":[{\"internalType\":\"uint256\",\"name\":\"range\",\"type\":\"uint256\"}],\"name\":\"functionWithKeywordParameter\",\"outputs\":[],\"stateMutability\":\"pure\",\"type\":\"function\"}]"
        
        	
        
        	
        		// RangeKeywordBin is the compiled bytecode used for deploying new contracts.
        		var RangeKeywordBin = "0x608060405234801561001057600080fd5b5060dc8061001f6000396000f3fe6080604052348015600f57600080fd5b506004361060285760003560e01c8063527a119f14602d575b600080fd5b60436004803603810190603f9190605b565b6045565b005b50565b6000813590506055816092565b92915050565b600060208284031215606e57606d608d565b5b6000607a848285016048565b91505092915050565b6000819050919050565b600080fd5b6099816083565b811460a357600080fd5b5056fea2646970667358221220d4f4525e2615516394055d369fb17df41c359e5e962734f27fd683ea81fd9db164736f6c63430008070033"
        
        		// DeployRangeKeyword deploys a new Ethereum contract, binding an instance of RangeKeyword to it.
        		func DeployRangeKeyword(auth *bind.TransactOpts, backend bind.ContractBackend ) (libcommon.Address, types.Transaction, *RangeKeyword, error) {
        		  parsed, err := abi.JSON(strings.NewReader(RangeKeywordABI))
        		  if err != nil {
        		    return libcommon.Address{}, nil, nil, err
        		  }
        		  
        		  address, tx, contract, err := bind.DeployContract(auth, parsed, libcommon.FromHex(RangeKeywordBin), backend )
        		  if err != nil {
        		    return libcommon.Address{}, nil, nil, err
        		  }
        		  return address, tx, &RangeKeyword{ RangeKeywordCaller: RangeKeywordCaller{contract: contract}, RangeKeywordTransactor: RangeKeywordTransactor{contract: contract}, RangeKeywordFilterer: RangeKeywordFilterer{contract: contract} }, nil
        		}
        	
        
        	// RangeKeyword is an auto generated Go binding around an Ethereum contract.
        	type RangeKeyword struct {
        	  RangeKeywordCaller     // Read-only binding to the contract
        	  RangeKeywordTransactor // Write-only binding to the contract
        	  RangeKeywordFilterer   // Log filterer for contract events
        	}
        
        	// RangeKeywordCaller is an auto generated read-only Go binding around an Ethereum contract.
        	type RangeKeywordCaller struct {
        	  contract *bind.BoundContract // Generic contract wrapper for the low level calls
        	}
        
        	// RangeKeywordTransactor is an auto generated write-only Go binding around an Ethereum contract.
        	type RangeKeywordTransactor struct {
        	  contract *bind.BoundContract // Generic contract wrapper for the low level calls
        	}
        
        	// RangeKeywordFilterer is an auto generated log filtering Go binding around an Ethereum contract events.
        	type RangeKeywordFilterer struct {
        	  contract *bind.BoundContract // Generic contract wrapper for the low level calls
        	}
        
        	// RangeKeywordSession is an auto generated Go binding around an Ethereum contract,
        	// with pre-set call and transact options.
        	type RangeKeywordSession struct {
        	  Contract     *RangeKeyword        // Generic contract binding to set the session for
        	  CallOpts     bind.CallOpts     // Call options to use throughout this session
        	  TransactOpts bind.TransactOpts // Transaction auth options to use throughout this session
        	}
        
        	// RangeKeywordCallerSession is an auto generated read-only Go binding around an Ethereum contract,
        	// with pre-set call options.
        	type RangeKeywordCallerSession struct {
        	  Contract *RangeKeywordCaller // Generic contract caller binding to set the session for
        	  CallOpts bind.CallOpts    // Call options to use throughout this session
        	}
        
        	// RangeKeywordTransactorSession is an auto generated write-only Go binding around an Ethereum contract,
        	// with pre-set transact options.
        	type RangeKeywordTransactorSession struct {
        	  Contract     *RangeKeywordTransactor // Generic contract transactor binding to set the session for
        	  TransactOpts bind.TransactOpts    // Transaction auth options to use throughout this session
        	}
        
        	// RangeKeywordRaw is an auto generated low-level Go binding around an Ethereum contract.
        	type RangeKeywordRaw struct {
        	  Contract *RangeKeyword // Generic contract binding to access the raw methods on
        	}
        
        	// RangeKeywordCallerRaw is an auto generated low-level read-only Go binding around an Ethereum contract.
        	type RangeKeywordCallerRaw struct {
        		Contract *RangeKeywordCaller // Generic read-only contract binding to access the raw methods on
        	}
        
        	// RangeKeywordTransactorRaw is an auto generated low-level write-only Go binding around an Ethereum contract.
        	type RangeKeywordTransactorRaw struct {
        		Contract *RangeKeywordTransactor // Generic write-only contract binding to access the raw methods on
        	}
        
        	// NewRangeKeyword creates a new instance of RangeKeyword, bound to a specific deployed contract.
        	func NewRangeKeyword(address libcommon.Address, backend bind.ContractBackend) (*RangeKeyword, error) {
        	  contract, err := bindRangeKeyword(address, backend, backend, backend)
        	  if err != nil {
        	    return nil, err
        	  }
        	  return &RangeKeyword{ RangeKeywordCaller: RangeKeywordCaller{contract: contract}, RangeKeywordTransactor: RangeKeywordTransactor{contract: contract}, RangeKeywordFilterer: RangeKeywordFilterer{contract: contract} }, nil
        	}
        
        	// NewRangeKeywordCaller creates a new read-only instance of RangeKeyword, bound to a specific deployed contract.
        	func NewRangeKeywordCaller(address libcommon.Address, caller bind.ContractCaller) (*RangeKeywordCaller, error) {
        	  contract, err := bindRangeKeyword(address, caller, nil, nil)
        	  if err != nil {
        	    return nil, err
        	  }
        	  return &RangeKeywordCaller{contract: contract}, nil
        	}
        
        	// NewRangeKeywordTransactor creates a new write-only instance of RangeKeyword, bound to a specific deployed contract.
        	func NewRangeKeywordTransactor(address libcommon.Address, transactor bind.ContractTransactor) (*RangeKeywordTransactor, error) {
        	  contract, err := bindRangeKeyword(address, nil, transactor, nil)
        	  if err != nil {
        	    return nil, err
        	  }
        	  return &RangeKeywordTransactor{contract: contract}, nil
        	}
        
        	// NewRangeKeywordFilterer creates a new log filterer instance of RangeKeyword, bound to a specific deployed contract.
         	func NewRangeKeywordFilterer(address libcommon.Address, filterer bind.ContractFilterer) (*RangeKeywordFilterer, error) {
         	  contract, err := bindRangeKeyword(address, nil, nil, filterer)
         	  if err != nil {
         	    return nil, err
         	  }
         	  return &RangeKeywordFilterer{contract: contract}, nil
         	}
        
        	// bindRangeKeyword binds a generic wrapper to an already deployed contract.
        	func bindRangeKeyword(address libcommon.Address, caller bind.ContractCaller, transactor bind.ContractTransactor, filterer bind.ContractFilterer) (*bind.BoundContract, error) {
        	  parsed, err := abi.JSON(strings.NewReader(RangeKeywordABI))
        	  if err != nil {
        	    return nil, err
        	  }
        	  return bind.NewBoundContract(address, parsed, caller, transactor, filterer), nil
        	}
        
        	// Call invokes the (constant) contract method with params as input values and
        	// sets the output to result. The result type might be a single field for simple
        	// returns, a slice of interfaces for anonymous returns and a struct for named
        	// returns.
        	func (_RangeKeyword *RangeKeywordRaw) Call(opts *bind.CallOpts, result *[]interface{}, method string, params ...interface{}) error {
        		return _RangeKeyword.Contract.RangeKeywordCaller.contract.Call(opts, result, method, params...)
        	}
        
        	// Transfer initiates a plain transaction to move funds to the contract, calling
        	// its default method if one is available.
        	func (_RangeKeyword *RangeKeywordRaw) Transfer(opts *bind.TransactOpts) (types.Transaction, error) {
        		return _RangeKeyword.Contract.RangeKeywordTransactor.contract.Transfer(opts)
        	}
        
        	// Transact invokes the (paid) contract method with params as input values.
        	func (_RangeKeyword *RangeKeywordRaw) Transact(opts *bind.TransactOpts, method string, params ...interface{}) (types.Transaction, error) {
        		return _RangeKeyword.Contract.RangeKeywordTransactor.contract.Transact(opts, method, params...)
        	}
        
        	// Call invokes the (constant) contract method with params as input values and
        	// sets the output to result. The result type might be a single field for simple
        	// returns, a slice of interfaces for anonymous returns and a struct for named
        	// returns.
        	func (_RangeKeyword *RangeKeywordCallerRaw) Call(opts *bind.CallOpts, result *[]interface{}, method string, params ...interface{}) error {
        		return _RangeKeyword.Contract.contract.Call(opts, result, method, params...)
        	}
        
        	// Transfer initiates a plain transaction to move funds to the contract, calling
        	// its default method if one is available.
        	func (_RangeKeyword *RangeKeywordTransactorRaw) Transfer(opts *bind.TransactOpts) (types.Transaction, error) {
        		return _RangeKeyword.Contract.contract.Transfer(opts)
        	}
        
        	// Transact invokes the (paid) contract method with params as input values.
        	func (_RangeKeyword *RangeKeywordTransactorRaw) Transact(opts *bind.TransactOpts, method string, params ...interface{}) (types.Transaction, error) {
        		return _RangeKeyword.Contract.contract.Transact(opts, method, params...)
        	}
        
        	
        		// FunctionWithKeywordParameter is a free data retrieval call binding the contract method 0x527a119f.
        		//
        		// Solidity: function functionWithKeywordParameter(uint256 range) pure returns()
        		func (_RangeKeyword *RangeKeywordCaller) FunctionWithKeywordParameter(opts *bind.CallOpts , range *big.Int ) ( error) {
        			var out []interface{}
        			err := _RangeKeyword.contract.Call(opts, &out, "functionWithKeywordParameter" , range)
        			
        			if err != nil {
        				return  err
        			}
        			
        			
        			return  err
        			
        		}
        
        		// FunctionWithKeywordParameter is a free data retrieval call binding the contract method 0x527a119f.
        		//
        		// Solidity: function functionWithKeywordParameter(uint256 range) pure returns()
        		func (_RangeKeyword *RangeKeywordSession) FunctionWithKeywordParameter( range *big.Int ) (   error) {
        		  return _RangeKeyword.Contract.FunctionWithKeywordParameter(&_RangeKeyword.CallOpts , range)
        		}
        
        		// FunctionWithKeywordParameter is a free data retrieval call binding the contract method 0x527a119f.
        		//
        		// Solidity: function functionWithKeywordParameter(uint256 range) pure returns()
        		func (_RangeKeyword *RangeKeywordCallerSession) FunctionWithKeywordParameter( range *big.Int ) (   error) {
        		  return _RangeKeyword.Contract.FunctionWithKeywordParameter(&_RangeKeyword.CallOpts , range)
        		}




        
--- FAIL: TestGolangBindings (0.15s)


FAIL
```
