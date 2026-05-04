##  33 Module introduction
- What to & Not To Test
	- What Should be & Shouldn't be tested
	- Good test are short and concise
	- Testing & Code Improvements work together Iteratively\

	<hr>
##  34 what to test and what not to 
- don't test things we can't change like third party module
	- example is the query selector
		- like if the query selector is returning the right
	- or the node module which is we cannot change
	- we can test: if there is form selector on UI 
	- don't test backend code implicitly in frontend test it in backend it self
	- things to test in frontend
		- reaction to different responses & errors

<hr> 

##  35 Writing Good Tests
- use AAA pattern
- Test only one thing/ behaviour 
- Focus on essence of the thing we are testing
	- example:
```js
it('should summarize all numbers values in an arrary')
 const numbers = [1,2] **
 
 const result = add(numbers);
 
 expect(result).toBe(expectedResult);
```
-  ** In above code we can just check with two numbers without complicating the test
- Keep your number of exceptions  (assertion) low
## 36 Only test one Thing

- A test should only test one feature
	- Example: 
		- say you have a function that does two things (this should is so wrong to do) validate input or transform it.
		- We don't want to put this things in same test; The test should only check either validation or transformation
			- in this too we only should validate if a string for being empty or having enough characters in different tests
		- Make it as granular  as possible

<hr>

##  37 Splitting functions for Easier testing & Better Code

- There is not much to write but we can see it in following example (we have this functionality every where in our code so making it granular will help all of us out):
- this is my own code: 
	- see generateInvoicePDF() in invoicePage.jsx line 180- 570
	- the generateInvoicePDF() can be written as following
```js
const generateInvoicePDF = async (invoice) => {

if (!invoice) {

console.error('generateInvoicePDF: invoice is undefined or null');

dispatch(setAlert('Unable to generate PDF: Invoice data is missing', 'error'));

return;

}

  


// the fecth Billing address can be written as the follwing whcih will make it easier to write test on
const userBillingAddress = await fetchUserBillingAddress()

  

// the code ranging from 204 - 213 can be written as following:
 definePDFStructure

try {

// Try to load image from public folder

const img = new Image();

await new Promise((resolve, reject) => {

const timeout = setTimeout(() => reject(new Error('Logo loading timeout')), 3000);

img.onload = () => {

clearTimeout(timeout);

try {

// Calculate logo dimensions (max height 18mm, maintain aspect ratio)

const maxLogoHeight = 18;

const logoAspectRatio = img.width / img.height;

const logoHeight = maxLogoHeight;

const logoWidth = logoHeight * logoAspectRatio;

// Add logo to PDF (right side, top)

doc.addImage(img, 'PNG', pageWidth - 14 - logoWidth, 8, logoWidth, logoHeight);

logoAdded = true;

resolve();

} catch (addImageError) {

console.warn('Error adding image to PDF:', addImageError);

reject(addImageError);

}

};

img.onerror = () => {

clearTimeout(timeout);

reject(new Error('Failed to load logo image'));

};

// Try without CORS first (same origin)

img.src = logoUrl;

});

} catch (logoError) {

console.warn('Could not load logo image, using text fallback:', logoError);

}

let yPos = 20;

  

// Top section: Logo on right, Invoice title and company name on left

// Left side: Invoice title and company name with better spacing

doc.setTextColor(0, 0, 0);

doc.setFontSize(36);

doc.setFont('helvetica', 'bold');

doc.text('INVOICE', 14, yPos);

doc.setFontSize(11);

doc.setFont('helvetica', 'bold');

doc.text('Spaced Revision Pvt Ltd', 14, yPos + 10);

// Right side: Text fallback if logo couldn't be loaded

if (!logoAdded) {

doc.setFontSize(10);

doc.setFont('helvetica', 'normal');

doc.setTextColor(...brandColor);

doc.text('spaced revision', pageWidth - 14, yPos + 5, { align: 'right' });

}

yPos += 22;

  

// Teal bar with invoice number and date - improved styling

doc.setFillColor(...brandColor);

const barHeight = 10;

doc.rect(14, yPos - 6, pageWidth - 28, barHeight, 'F');

doc.setTextColor(255, 255, 255);

doc.setFontSize(11);

doc.setFont('helvetica', 'bold');

// Invoice number on left - improved format (remove INV- prefix, keep the number part)

let invoiceNum = 'N/A';

if (invoice.invoice_number) {

// Extract just the number part after INV- (e.g., "20260109-00009")

invoiceNum = invoice.invoice_number.replace(/^INV-/, '');

}

doc.text(`NO. ${invoiceNum}`, 18, yPos);

// Date on right - better formatting (e.g., "9 January 2026")

let invoiceDate = 'N/A';

if (invoice.invoice_date) {

const date = new Date(invoice.invoice_date);

const months = ['January', 'February', 'March', 'April', 'May', 'June',

'July', 'August', 'September', 'October', 'November', 'December'];

invoiceDate = `${date.getDate()} ${months[date.getMonth()]} ${date.getFullYear()}`;

}

doc.text(`Date: ${invoiceDate}`, pageWidth - 18, yPos, { align: 'right' });

yPos += 16;

  

// Two-column layout: Billed to (left) and From (right) - improved spacing

const leftColX = 14;

const rightColX = pageWidth / 2 + 10;

const colWidth = (pageWidth - 28) / 2 - 10;

  

// Left column: Billed to - improved styling

doc.setTextColor(...brandColor);

doc.setFontSize(12);

doc.setFont('helvetica', 'bold');

doc.text('Billed to:', leftColX, yPos);

doc.setFontSize(10);

doc.setFont('helvetica', 'normal');

doc.setTextColor(0, 0, 0);

// Format billing address

let addressLines = [];

if (billingAddress) {

if (billingAddress.line1) addressLines.push(billingAddress.line1);

if (billingAddress.line2) addressLines.push(billingAddress.line2);

const cityStatePincode = [

billingAddress.city,

billingAddress.state,

billingAddress.pincode

].filter(Boolean).join(', ');

if (cityStatePincode) addressLines.push(cityStatePincode);

if (billingAddress.country && billingAddress.country !== 'India') {

addressLines.push(billingAddress.country);

}

}

const formattedAddress = addressLines.length > 0 ? addressLines.join(', ') : '';

const billedToInfo = [

['Name:', invoice.user_name || ''],

['Address:', formattedAddress],

['Email Id:', invoice.user_email || ''],

];

  

let billedToY = yPos + 8;

billedToInfo.forEach(([label, value]) => {

doc.setFont('helvetica', 'bold');

doc.setTextColor(70, 70, 70);

doc.setFontSize(9);

doc.text(label, leftColX, billedToY);

if (value) {

doc.setFont('helvetica', 'normal');

doc.setTextColor(0, 0, 0);

const lines = doc.splitTextToSize(value, colWidth - 18);

doc.text(lines, leftColX + 18, billedToY);

billedToY += Math.max(lines.length * 4.5, 5);

} else {

billedToY += 5;

}

});

  

// Right column: From - improved styling

doc.setFontSize(12);

doc.setFont('helvetica', 'bold');

doc.setTextColor(...brandColor);

doc.text('From:', rightColX, yPos);

doc.setFontSize(10);

doc.setFont('helvetica', 'normal');

doc.setTextColor(0, 0, 0);

const companyInfo = [

'Spaced Revision Pvt Ltd,',

'Head Office: Gairi Goan Tadong, 737102, Gangtok, Sikkim.',

'RO: HD-267, Block L, WeWork Embassy TechVillag,',

'Bellandur, Bangalore South, Bangalore-560103, Karnataka',

'GST: 11ABNCS7455G1Z5'

];

  

let fromY = yPos + 8;

companyInfo.forEach((line) => {

const lines = doc.splitTextToSize(line, colWidth - 10);

doc.text(lines, rightColX, fromY);

fromY += lines.length * 5;

});

  

// Set yPos to the maximum of both columns with better spacing

yPos = Math.max(billedToY, fromY) + 15;

  

// Course Details Table

const tableData = [];

if (invoice.invoice_type === 'conversation credit' || invoice.invoice_type === 'conversation_credits' || invoice.invoice_type === 'CONVERSATION_CREDITS') {

tableData.push([

'Conversation Credits',

'1',

formatCurrencyForPDF(invoice.base_amount || invoice.final_amount_paid, invoice.currency),

formatCurrencyForPDF(invoice.discount_amount || 0, invoice.currency),

]);

} else if (invoice.course_name) {

tableData.push([

invoice.course_name.length > 30 ? invoice.course_name.substring(0, 30) + '...' : invoice.course_name,

'1',

formatCurrencyForPDF(invoice.base_amount || invoice.course_original_price || 0, invoice.currency),

formatCurrencyForPDF(invoice.discount_amount || 0, invoice.currency),

]);

} else {

const paymentType = isEMIPayment(invoice) ? 'EMI Payment' : formatInvoiceType(invoice.invoice_type);

tableData.push([

paymentType.length > 30 ? paymentType.substring(0, 30) + '...' : paymentType,

'1',

formatCurrencyForPDF(invoice.base_amount || 0, invoice.currency),

formatCurrencyForPDF(invoice.discount_amount || 0, invoice.currency),

]);

}

  

autoTable(doc, {

startY: yPos,

head: [['Course', 'Quantity', 'Price', 'Discount']],

body: tableData,

theme: 'striped',

headStyles: {

fillColor: brandColor,

textColor: [255, 255, 255],

fontStyle: 'bold',

fontSize: 11,

halign: 'left',

cellPadding: 5

},

styles: {

fontSize: 10,

cellPadding: 5,

halign: 'left',

overflow: 'linebreak',

cellWidth: 'wrap',

lineColor: [200, 200, 200],

lineWidth: 0.1

},

alternateRowStyles: {

fillColor: [250, 250, 250]

},

columnStyles: {

0: { cellWidth: 85, halign: 'left', overflow: 'linebreak', fontStyle: 'normal' },

1: { cellWidth: 28, halign: 'center', fontStyle: 'normal' },

2: { cellWidth: 40, halign: 'right', fontStyle: 'normal' },

3: { cellWidth: 35, halign: 'right', fontStyle: 'normal' },

},

margin: { left: 14, right: 14, top: 2, bottom: 2 },

});

  

yPos = doc.lastAutoTable.finalY + 18;

  

// Summary Section - improved styling

// The final_amount_paid includes 18% GST, so we need to calculate backwards

// If final_amount_paid = base_amount + GST (18% of base_amount)

// final_amount_paid = base_amount * 1.18

// base_amount = final_amount_paid / 1.18

// GST = final_amount_paid - base_amount

const finalAmountPaid = parseFloat(invoice.final_amount_paid) || 0;

const gstRate = 0.18; // 18% GST

const baseAmount = finalAmountPaid / (1 + gstRate); // Amount before GST

const gstAmount = finalAmountPaid - baseAmount; // GST amount (18% of base amount)

  

// Right-align summary section - improved positioning

const summaryX = pageWidth - 75;

const summaryLabelX = pageWidth - 120;

doc.setFontSize(10);

doc.setFont('helvetica', 'normal');

doc.setTextColor(0, 0, 0);

// Subtotal (base amount before GST) - add label

doc.text('Subtotal:', summaryLabelX, yPos, { align: 'right' });

doc.setFont('helvetica', 'bold');

doc.text(formatCurrencyForPDF(baseAmount, invoice.currency), summaryX, yPos, { align: 'right' });

yPos += 7;

// GST - 18%

doc.setFont('helvetica', 'normal');

doc.text('GST:', summaryLabelX, yPos, { align: 'right' });

doc.setFont('helvetica', 'bold');

doc.text(formatCurrencyForPDF(gstAmount, invoice.currency), summaryX, yPos, { align: 'right' });

yPos += 10;

  

// Total (with underline effect and better styling)

doc.setDrawColor(...brandColor);

doc.setLineWidth(0.8);

doc.line(summaryLabelX - 10, yPos - 3, summaryX + 5, yPos - 3);

doc.setFontSize(12);

doc.setFont('helvetica', 'bold');

doc.setTextColor(...brandColor);

doc.text('Total:', summaryLabelX, yPos, { align: 'right' });

doc.setTextColor(0, 0, 0);

doc.text(formatCurrencyForPDF(finalAmountPaid, invoice.currency), summaryX, yPos, { align: 'right' });

yPos += 18;

  

// Payment Information Section - improved styling

doc.setFontSize(11);

doc.setFont('helvetica', 'bold');

doc.setTextColor(...brandColor);

doc.text('Payment Information:', 14, yPos);

yPos += 9;

doc.setFontSize(10);

doc.setFont('helvetica', 'normal');

doc.setTextColor(0, 0, 0);

// Payment status with better formatting

const paymentStatus = (invoice.payment_status || 'N/A').toUpperCase();

doc.setFont('helvetica', 'bold');

doc.setTextColor(70, 70, 70);

doc.text('Status:', 14, yPos);

doc.setFont('helvetica', 'normal');

doc.setTextColor(0, 0, 0);

doc.text(paymentStatus, 45, yPos);

yPos += 6;

// Order ID if available

if (invoice.razorpay_order_id) {

doc.setFont('helvetica', 'bold');

doc.setTextColor(70, 70, 70);

doc.text('Order ID:', 14, yPos);

doc.setFont('helvetica', 'normal');

doc.setTextColor(0, 0, 0);

const orderIdLines = doc.splitTextToSize(invoice.razorpay_order_id, pageWidth - 65);

doc.text(orderIdLines, 50, yPos);

yPos += orderIdLines.length * 5.5;

}

// Payment ID if available

if (invoice.razorpay_payment_id) {

doc.setFont('helvetica', 'bold');

doc.setTextColor(70, 70, 70);

doc.text('Payment ID:', 14, yPos);

doc.setFont('helvetica', 'normal');

doc.setTextColor(0, 0, 0);

const paymentIdLines = doc.splitTextToSize(invoice.razorpay_payment_id, pageWidth - 65);

doc.text(paymentIdLines, 55, yPos);

yPos += paymentIdLines.length * 5.5;

}

  

// Ensure footer is properly positioned

yPos = Math.max(yPos + 10, pageHeight - 40);

  

// Footer - improved spacing and styling

doc.setFontSize(10);

doc.setFont('helvetica', 'italic');

doc.setTextColor(100, 100, 100);

doc.text('Thank you for choosing us! Happy Learning', pageWidth / 2, yPos, { align: 'center' });

yPos += 10;

// Footer bar with website - improved styling

doc.setFillColor(...brandColor);

const footerBarHeight = 8;

doc.rect(0, yPos - 4, pageWidth, footerBarHeight, 'F');

doc.setTextColor(255, 255, 255);

doc.setFontSize(9);

doc.setFont('helvetica', 'normal');

doc.text('www.spacedrevision.com', 18, yPos + 3);

  

// Save PDF

doc.save(`Invoice-${invoice.invoice_number || invoice.invoice_id}.pdf`);

} catch (error) {

console.error('Error generating PDF:', error);

dispatch(setAlert('Failed to generate PDF. Please try again.', 'error'));

}

};
```
