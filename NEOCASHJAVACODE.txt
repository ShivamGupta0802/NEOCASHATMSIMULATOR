import javax.swing.*;
import javax.swing.text.*;
import java.awt.*;
import java.awt.event.*;
import java.io.*;
import java.util.*;
import java.text.SimpleDateFormat;

// Model: represents a user account
class AccountRecord implements Serializable {
    private String holder;
    private int pin;
    private double funds;
    private LinkedList<Ledger> ledger = new LinkedList<>();

    public AccountRecord(String holder, int pin, double funds) {
        this.holder = holder;
        this.pin = pin;
        this.funds = funds;
    }

    public String getHolder() { return holder; }
    public int getPin() { return pin; }
    public double getFunds() { return funds; }

    public void deposit(double amt) {
        if (amt <= 0) throw new IllegalArgumentException("Invalid deposit.");
        funds += amt;
        record("Deposit", amt);
    }

    public void withdraw(double amt) {
        if (amt <= 0) throw new IllegalArgumentException("Invalid withdrawal.");
        if (amt > funds) throw new IllegalArgumentException("Insufficient balance.");
        funds -= amt;
        record("Withdrawal", amt);
    }

    private void record(String type, double amt) {
        ledger.addFirst(new Ledger(type, amt, funds));
        if (ledger.size() > 6) ledger.removeLast();
    }

    public String viewLedger() {
        if (ledger.isEmpty()) return "No transactions found.";
        StringBuilder sb = new StringBuilder("Recent Transactions:\n");
        for (Ledger l : ledger) sb.append(l).append("\n");
        return sb.toString();
    }
}

// Ledger entry class
class Ledger implements Serializable {
    private String action;
    private double amount;
    private double resultingBalance;
    private String timestamp;

    public Ledger(String action, double amount, double resultingBalance) {
        this.action = action;
        this.amount = amount;
        this.resultingBalance = resultingBalance;
        this.timestamp = new SimpleDateFormat("dd/MM/yyyy HH:mm:ss").format(new Date());
    }

    @Override
    public String toString() {
        return String.format("[%s] %s Rs.%.2f -> Balance Rs.%.2f",
                timestamp, action, amount, resultingBalance);
    }
}

// Data manager for persistence
class DataVault {
    private static final String FILE = "atmdata.bin";

    @SuppressWarnings("unchecked")
    public static Map<Integer, AccountRecord> loadAll() {
        try (ObjectInputStream in = new ObjectInputStream(new FileInputStream(FILE))) {
            return (Map<Integer, AccountRecord>) in.readObject();
        } catch (Exception e) {
            return new HashMap<>();
        }
    }

    public static void saveAll(Map<Integer, AccountRecord> data) {
        try (ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(FILE))) {
            out.writeObject(data);
        } catch (IOException ignored) {}
    }
}

// Controller & GUI combined
public class NeoCashATM extends JFrame {
    private CardLayout layout = new CardLayout();
    private JPanel container = new JPanel(layout);
    private Map<Integer, AccountRecord> records;
    private AccountRecord current;

    // Components reused across screens
    private JTextField amountBox;
    private JLabel balanceDisplay;

    // Color theme
    private final Color BG = new Color(18, 24, 34);
    private final Color PANEL_BG = new Color(24, 32, 44);
    private final Color ACCENT = new Color(16, 185, 129);
    private final Color BUTTON_BG = new Color(30, 41, 59);
    private final Color TEXT_BG = new Color(40, 52, 68);
    private final Color FG = Color.WHITE;

    // Constructor
    public NeoCashATM() {
        // Global UI theme setup
        UIManager.put("Label.foreground", FG);
        UIManager.put("Button.foreground", FG);
        UIManager.put("Button.background", BUTTON_BG);
        UIManager.put("Panel.background", PANEL_BG);
        UIManager.put("TextField.background", TEXT_BG);
        UIManager.put("TextField.foreground", FG);
        UIManager.put("PasswordField.background", TEXT_BG);
        UIManager.put("PasswordField.foreground", FG);

        // Customize JOptionPane dialogs (dark theme)
        UIManager.put("OptionPane.background", PANEL_BG);
        UIManager.put("OptionPane.messageForeground", FG);
        UIManager.put("OptionPane.messageFont", new Font("SansSerif", Font.PLAIN, 14));
        UIManager.put("OptionPane.border", BorderFactory.createLineBorder(ACCENT, 2));
        UIManager.put("Button.background", ACCENT);
        UIManager.put("Button.foreground", Color.WHITE);
        UIManager.put("Button.select", ACCENT);
        UIManager.put("Button.focus", ACCENT);

        records = DataVault.loadAll();
        if (records.isEmpty()) {
            records.put(1001, new AccountRecord("Riya Sharma", 1001, 15000));
            records.put(2002, new AccountRecord("Karan Patel", 2002, 10000));
            DataVault.saveAll(records);
        }

        buildScreens();
        getContentPane().setBackground(BG);
        setTitle("NeoCash ATM System");
        setDefaultCloseOperation(EXIT_ON_CLOSE);
        setSize(500, 400);
        setLocationRelativeTo(null);
        setVisible(true);
    }

    private void buildScreens() {
        container.setBackground(PANEL_BG);
        container.add(homePanel(), "home");
        container.add(loginPanel(), "login");
        container.add(registerPanel(), "register");
        container.add(dashboardPanel(), "dashboard");
        add(container);
        layout.show(container, "home");
    }

    private JPanel homePanel() {
        JPanel p = new JPanel(new GridLayout(3, 1, 8, 8));
        p.setBackground(PANEL_BG);
        p.setBorder(BorderFactory.createEmptyBorder(20, 20, 20, 20));

        JLabel title = new JLabel("Welcome to NeoCash ATM", SwingConstants.CENTER);
        title.setFont(new Font("SansSerif", Font.BOLD, 20));
        title.setForeground(FG);

        JButton login = new JButton("Sign In");
        JButton create = new JButton("Open New Account");
        styleButton(login);
        styleButton(create);

        login.addActionListener(e -> layout.show(container, "login"));
        create.addActionListener(e -> layout.show(container, "register"));

        p.add(title);
        p.add(login);
        p.add(create);
        return p;
    }

    private JPanel loginPanel() {
        JPanel p = new JPanel(new GridLayout(4, 1, 8, 8));
        p.setBackground(PANEL_BG);
        p.setBorder(BorderFactory.createEmptyBorder(20, 40, 20, 40));

        JLabel lbl = new JLabel("Enter 4-digit PIN:", SwingConstants.CENTER);
        lbl.setForeground(FG);
        JPasswordField field = new JPasswordField();
        styleTextField(field);

        JButton loginBtn = new JButton("Access Account");
        JButton back = new JButton("Back");
        styleButton(loginBtn);
        styleButton(back);

        loginBtn.addActionListener(e -> {
            try {
                int pin = Integer.parseInt(new String(field.getPassword()));
                if (records.containsKey(pin)) {
                    current = records.get(pin);
                    refreshDashboard();
                    layout.show(container, "dashboard");
                } else {
                    JOptionPane.showMessageDialog(this, "Invalid PIN.");
                }
            } catch (NumberFormatException ex) {
                JOptionPane.showMessageDialog(this, "Enter numeric PIN only.");
            }
        });

        back.addActionListener(e -> layout.show(container, "home"));
        p.add(lbl); p.add(field); p.add(loginBtn); p.add(back);
        return p;
    }

    private JPanel registerPanel() {
        JPanel p = new JPanel(new GridLayout(7, 1, 5, 5));
        p.setBackground(PANEL_BG);
        p.setBorder(BorderFactory.createEmptyBorder(12, 30, 12, 30));

        JTextField name = new JTextField();
        styleTextField(name);
        JPasswordField pin = new JPasswordField();
        styleTextField(pin);
        JTextField init = new JTextField();
        styleTextField(init);

        JButton create = new JButton("Create Account");
        JButton back = new JButton("Back");
        styleButton(create);
        styleButton(back);

        p.add(centerLabel("Full Name:")); p.add(name);
        p.add(centerLabel("Choose a 4-digit PIN:")); p.add(pin);
        p.add(centerLabel("Initial Deposit (Rs):")); p.add(init);
        p.add(create); p.add(back);

        create.addActionListener(e -> {
            try {
                String nm = name.getText().trim();
                int pn = Integer.parseInt(new String(pin.getPassword()));
                double amt = Double.parseDouble(init.getText());
                if (nm.isEmpty() || String.valueOf(pn).length() != 4 || amt <= 0)
                    throw new IllegalArgumentException("Invalid details.");

                if (records.containsKey(pn)) {
                    JOptionPane.showMessageDialog(this, "PIN already exists.");
                    return;
                }

                records.put(pn, new AccountRecord(nm, pn, amt));
                DataVault.saveAll(records);
                JOptionPane.showMessageDialog(this, "Account created successfully!");
                layout.show(container, "home");
            } catch (Exception ex) {
                JOptionPane.showMessageDialog(this, ex.getMessage());
            }
        });

        back.addActionListener(e -> layout.show(container, "home"));
        return p;
    }

    private JPanel dashboardPanel() {
        JPanel p = new JPanel(new GridLayout(8, 1, 8, 8));
        p.setBackground(PANEL_BG);
        p.setBorder(BorderFactory.createEmptyBorder(10, 10, 10, 10));

        JLabel welcome = new JLabel("", SwingConstants.CENTER);
        welcome.setFont(new Font("SansSerif", Font.BOLD, 16));
        welcome.setForeground(FG);
        balanceDisplay = new JLabel("", SwingConstants.CENTER);
        balanceDisplay.setForeground(FG);
        amountBox = new JTextField();
        styleTextField(amountBox);

        JButton deposit = new JButton("Deposit");
        JButton withdraw = new JButton("Withdraw");
        JButton balance = new JButton("Show Balance");
        JButton history = new JButton("Show Transactions");
        JButton logout = new JButton("Logout");

        styleButton(deposit);
        styleButton(withdraw);
        styleButton(balance);
        styleButton(history);
        styleButton(logout);

        deposit.addActionListener(e -> transact("deposit"));
        withdraw.addActionListener(e -> transact("withdraw"));
        balance.addActionListener(e -> JOptionPane.showMessageDialog(this,
                "Available Balance: Rs." + current.getFunds()));
        history.addActionListener(e -> JOptionPane.showMessageDialog(this,
                current.viewLedger(), "Transaction Log", JOptionPane.INFORMATION_MESSAGE));
        logout.addActionListener(e -> layout.show(container, "home"));

        p.add(welcome);
        p.add(balanceDisplay);
        p.add(amountBox);
        p.add(deposit);
        p.add(withdraw);
        p.add(balance);
        p.add(history);
        p.add(logout);

        p.putClientProperty("welcome", welcome);
        return p;
    }

    private void refreshDashboard() {
        Component dashboard = container.getComponent(3);
        if (dashboard instanceof JPanel) {
            Component[] comps = ((JPanel) dashboard).getComponents();
            for (Component comp : comps) {
                if (comp instanceof JLabel && ((JLabel) comp).getText().isEmpty()) {
                    ((JLabel) comp).setText("Welcome, " + current.getHolder());
                    break;
                }
            }
        }
        balanceDisplay.setText("Balance: Rs." + current.getFunds());
        amountBox.setText("");
    }

    private void transact(String type) {
        try {
            double amt = Double.parseDouble(amountBox.getText());
            if (type.equals("deposit")) {
                current.deposit(amt);
                JOptionPane.showMessageDialog(this, "Amount deposited.");
            } else {
                current.withdraw(amt);
                JOptionPane.showMessageDialog(this, "Withdrawal successful.");
            }
            DataVault.saveAll(records);
            refreshDashboard();
        } catch (NumberFormatException ex) {
            JOptionPane.showMessageDialog(this, "Enter numeric amount.");
        } catch (IllegalArgumentException ex) {
            JOptionPane.showMessageDialog(this, ex.getMessage());
        }
    }

    // UI styling helpers
    private void styleButton(AbstractButton b) {
        b.setBackground(ACCENT);
        b.setForeground(FG);
        b.setFocusPainted(false);
        b.setBorder(BorderFactory.createEmptyBorder(8, 12, 8, 12));
        b.setOpaque(true);
    }

    private void styleTextField(JTextComponent tf) {
        tf.setBackground(TEXT_BG);
        tf.setForeground(FG);
        tf.setCaretColor(FG);
        tf.setBorder(BorderFactory.createCompoundBorder(
                BorderFactory.createLineBorder(BUTTON_BG, 1),
                BorderFactory.createEmptyBorder(6, 6, 6, 6)
        ));
        tf.setOpaque(true);
    }

    private JLabel centerLabel(String text) {
        JLabel l = new JLabel(text, SwingConstants.CENTER);
        l.setForeground(FG);
        return l;
    }

    public static void main(String[] args) {
        SwingUtilities.invokeLater(NeoCashATM::new);
    }
}