![image](picture/sign_state_1.png)

![image](picture/sign_state_3.png) It is the state of t+1 (account trie).

![image](picture/sign_state_4.png) It is a state transition function, which can also be understood as an execution engine.

![image](picture/sign_state_5.png) is a transaction

![image](picture/sign_state_6.png)

![image](picture/sign_state_7.png) Is a state transition function at the block level.

![image](picture/sign_state_8.png) It is a block and consists of many transactions.

![image](picture/sign_state_9.png) Transaction at position 0.

![image](picture/sign_state_10.png) Is the block termination state transition function (a function that rewards the miner).

![image](picture/sign_ether.png) Ether logo

![image](picture/sign_ether_value.png) The conversion relationship between the various units used in Ethereum and Wei (for example: a Finney corresponds to 10^15 Wei).

![image](picture/sign_machine_state.png) machine-state

## Some basic rules

- For most functions, they are identified by uppercase letters.
- Tuples are generally identified by capital letters
- A scalar or fixed-size array of bytes is identified by a lowercase letter. For example, n represents the nonce of the transaction, and there may be some exceptions, such as δ representing the amount of stack data required for a given instruction.
- Variable-length byte arrays generally use bold lowercase letters. For example, **o** represents the output data of a message call. It is also possible to use bold uppercase letters for some important things.

![image](picture/sign_set_b.png) Byte sequence
![image](picture/sign_set_p.png) positive integer
![image](picture/sign_set_b32.png) byte sequence length of 32 bytes

![image](picture/sign_set_p256.png) positive integer smaller than 2 ^ 256,
**[ ]** is used to index array elements which corresponds to
![image](picture/sign_stack.png) the first object represents a stack machine (machine's stack) is
![image](picture/sign_memory.png) representative of the machine memory (machine's memory) the first 32 elements inside
![image](picture/sign_placeholder_1.png) a placeholder, any character may be representative of any object

![image](picture/sign_placeholder_2.png) Value representative of the object to be modified
![image](picture/sign_placeholder_3.png) intermediate state
![image](picture/sign_placeholder_4.png) 中间状态 2
![image](picture/sign_func_1.png) ![image](picture/sign_func_2.png) intermediate state 2
if the previous f represents a function, f \* represents the back of a similar function, but a function f is sequentially executed for the inner elements.

![image](picture/sign_last_item.png) It represents the final element of the list which is
![image](picture/sign_last_item_1.png) representative of the last element of the list which is

![image](picture/sign_seq_item.png) seeking the length x

![image](picture/sign_state_nonce.png) a represents an address representing the hash of the root hash code of the nonce
![image](picture/sign_state_balance.png) banlance
![image](picture/sign_state_root.png) storage trie of root hash
![image](picture/sign_state_code.png) If code is b then KEC(b)=== this hash

![image](picture/sign_l1.png)

![image](picture/sign_ls.png) world state collapse function
![image](picture/sign_pa.png)

![image](picture/sign_math_any.png) any
![image](picture/sign_math_or.png) or
![image](picture/sign_math_and.png) and

![image](picture/sign_homestead.png) Homestead

## Transaction

![image](picture/sign_t_nonce.png) transaction nonce
![image](picture/sign_t_gasprice.png) gasPrice
![image](picture/sign_t_gaslimit.png) gasLimit
![image](picture/sign_t_to.png) to
![image](picture/sign_t_value.png) value

![image](picture/sign_t_w.png)![image](picture/sign_t_tr.png)![image](picture/sign_t_ts.png)The sender's address can be obtained by three values.

![image](picture/sign_t_ti.png) Initialization code contract
![image](picture/sign_t_data.png) method call
![image](picture/sign_t_lt.png)

## Block head

![image](picture/sign_h_p.png)ParentHash
![image](picture/sign_h_o.png)OmmersHash
![image](picture/sign_h_c.png)beneficiary miner address
![image](picture/sign_h_r.png)stateRoot
![image](picture/sign_h_t.png)transactionRoot
![image](picture/sign_h_e.png)receiptRoot
![image](picture/sign_h_b.png)logsBloom
![image](picture/sign_h_d.png)difficult
![image](picture/sign_h_i.png)height number
![image](picture/sign_h_l.png)gasLimit
![image](picture/sign_h_g.png)gasUsed
![image](picture/sign_h_s.png)timestamp
![image](picture/sign_h_x.png)extraData
![image](picture/sign_h_m.png)mixHash
![image](picture/sign_h_n.png)nonce

## Receipt

![image](picture/sign_r_i.png) Recitt of the i-th transaction

![image](picture/sign_receipt.png)
![image](picture/sign_r_state.png) World-state after execution of the transaction

![image](picture/sign_r_gasused.png)the total amount of gas after transaction execution block
![image](picture/sign_r_bloom.png)present transaction log to perform all the Bloom filter data generated
![image](picture/sign_r_log.png)set of logs generated by the transaction

![image](picture/sign_r_logentry.png) Log entry Oa log generated address, Ot topic Od time

## Transaction execution

![image](picture/sign_substate_a.png) substate
![image](picture/sign_substate_as.png) suicide set
![image](picture/sign_substate_al.png) log series
![image](picture/sign_substate_ar.png) refund balance

![image](picture/sign_gas_total.png) The total amount of gas used during the transaction.
![image](picture/sign_gas_log.png) The log generated by the transaction.

![image](picture/sign_i_a.png) Execute code owner
![image](picture/sign_i_o.png) originator of the transaction
![image](picture/sign_i_p.png) gasPrice
![image](picture/sign_i_d.png) inputdata
![image](picture/sign_i_s.png) cause the address to execute code, so if a transaction is the transaction originator
![image](picture/sign_i_v.png) value
![image](picture/sign_i_b.png) code needs to execute
![image](picture/sign_i_h.png) the current header area
![image](picture/sign_i_e.png) current call depth

![image](picture/sign_exec_model.png) Execution model **s** suicide set; **l** log collection **o** output; **r** refund

![image](picture/sign_exec_func.png) Execution function

![image](picture/sign_m_g.png) Currently available gas
![image](picture/sign_u_pc.png) program counter
![image](picture/sign_u_m.png) the memory contents of
![image](picture/sign_u_i.png) the memory word effective amount of
![image](picture/sign_u_s.png) stack contents

![image](picture/sign_m_w.png) w represents the current instruction that needs to be executed

![image](picture/sign_stack_removed.png) Stack Object number of instructions required to remove
![image](picture/sign_stack_added.png) the number of instructions required to increase the stack object
