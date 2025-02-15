// SPDX-License-Identifier: MIT
pragma solidity ^0.8.18;

// ERC-20 Interface
interface IERC20 {
    function transfer(address recipient, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function allowance(address owner, address spender) external view returns (uint256);
}

contract PayrollSystem {
    struct Employee {
        address wallet;
        uint256 salary;
        uint256 lastPaid;
        uint256 startTime;
        uint256 vestingDuration;
        bool exists;
    }

    mapping(address => Employee) public employees;
    address public owner;
    IERC20 public token; // ERC-20 Token (USDC, DAI, etc.)

    event EmployeeAdded(address indexed wallet, uint256 salary, uint256 vestingDuration);
    event SalaryPaid(address indexed employee, uint256 amount);
    
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this function");
        _;
    }

    constructor(address _token) {
        require(_token != address(0), "Invalid token address");
        owner = msg.sender;
        token = IERC20(_token);
    }

    function addEmployee(address _wallet, uint256 _salary, uint256 _vestingDuration) external onlyOwner {
        require(!employees[_wallet].exists, "Employee already exists");
        employees[_wallet] = Employee(_wallet, _salary, block.timestamp, block.timestamp, _vestingDuration, true);
        emit EmployeeAdded(_wallet, _salary, _vestingDuration);
    }

    function paySalary(address _employee) external onlyOwner {
        require(employees[_employee].exists, "Employee does not exist");
        Employee storage emp = employees[_employee];
        require(block.timestamp >= emp.lastPaid + 30 days, "Salary already paid this month");

        uint256 vestedAmount = (emp.salary * (block.timestamp - emp.startTime)) / emp.vestingDuration;
        require(vestedAmount > 0, "No vested salary available");
        require(token.balanceOf(address(this)) >= vestedAmount, "Insufficient funds in contract");

        emp.lastPaid = block.timestamp;
        require(token.transfer(_employee, vestedAmount), "Token transfer failed");

        emit SalaryPaid(_employee, vestedAmount);
    }

    function contractBalance() external view returns (uint256) {
        return token.balanceOf(address(this));
    }

    function fundPayroll(uint256 _amount) external onlyOwner {
        require(token.transferFrom(msg.sender, address(this), _amount), "Funding failed");
    }
}
# Decentralized-Payroll-System
