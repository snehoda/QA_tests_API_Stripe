# QA_tests_API_Stripe
API Stripe testing


const { expect } = require('chai');
const Invoice = require('../Model');
const { invoiceCreate, invoiceCheckout, invoiceGetById } = require('./requests');
const { clientCreate } = require('../../client/_test/request');
const { client } = require("../../client/_test/data");
const generateId = require("../../utils/generateId");
const { invoice, items } = require ('./data');

const invoiceCheckOutOKMessage = 'Stripe payment link';
const invoiceCheckOutGetErrorMessage = 'Invoice get error';
const invoiceCheckOutAmountZeroMessage = 'total amount due cannot be zero';
const invoiceCheckOutMissingPaymentMessage = 'Missing payment link URL';

before(() => {
  return Invoice.deleteMany({});
});

describe('Positive Tests', () => {
  let clientId = null;
  let invoiceId = null;

  before(`Create a new client with cookie as BUSINESSOWNER`, (done) => {
      clientCreate(client, process.env.COOKIE_BUSINESSOWNER.split(','))
        .end(
          (err, res) => {
            if (err) return done(err);
            clientId = res.body.payload.clientId;
            done();
          },
        );
    });

  before(`Create invoice with items - FOR POSITIVE TESTS`, (done) => {
      invoiceCreate(
        {
          ...invoice,
          client: { _id: clientId },
          invoiceStripeId: generateId(),
          items: items,
          total: 15
        },
        process.env.COOKIE_BUSINESSOWNER.split(','),
      )
        .expect(201)
        .end((err, res) => {
          if (err) return done(err);
          expect(res.body.success).to.be.true;
          invoiceId = res.body.payload.invoiceId;
          done();
        });
    });

  it(`Get Invoice By ID`, (done) => {
      invoiceGetById(
        invoiceId,
        process.env.COOKIE_BUSINESSOWNER.split(','),
      )
        .expect(200)
        .end((err, res) => {
          if (err) return done(err);
          expect(res.body.success).to.be.true;
          done();
        })
    });

  it(`A business owner should be able to checkout`, (done) => {

      invoiceCheckout(
        invoiceId,
        process.env.COOKIE_BUSINESSOWNER.split(','),
      )
        .expect(200)
        .end((err, res) => {
          if (err) return done(err);
          expect(res.body.success).true;
          expect(res.body.message).to.eq(invoiceCheckOutOKMessage)
          done();
        });
    })

})

describe('Negative Tests', () => {
  let clientId = null;
  let invoiceId = null;
  let invoiceIdTotalZero = null;
  let invoiceIdMissingPayment = null;

  before(`Create a new client with cookie as BUSINESSOWNER`, (done) => {
    clientCreate(client, process.env.COOKIE_BUSINESSOWNER.split(','))
      .end(
        (err, res) => {
          if (err) return done(err);
          clientId = res.body.payload.clientId;
          done();
        },
      );
  });

  before(`Create invoice with items`, (done) => {
    invoiceCreate(
      {
        ...invoice,
        client: { _id: clientId },
        invoiceStripeId: generateId(),
        items: items,
        total: 15
      },
      process.env.COOKIE_BUSINESSOWNER.split(','),
    )
      .expect(201)
      .end((err, res) => {
        if (err) return done(err);
        expect(res.body.success).to.be.true;
        invoiceId = res.body.payload.invoiceId;
        done();
      });
  });

  before(`Create invoice without with missing payments`, (done) => {
    invoiceCreate(
      {
        ...invoice,
        total: 15,
        items: items,
        client: { _id: clientId },
      },
      process.env.COOKIE_BUSINESSOWNER.split(','),
    )
      .expect(201)
      .end((err, res) => {
        if (err) return done(err);
        expect(res.body.success).to.be.true;
        invoiceIdMissingPayment = res.body.payload.invoiceId;
        done();
      });
  });

  before(`Create invoice without with amount Zero`, (done) => {
    invoiceCreate(
      {
        ...invoice,
        total: 0,
        items: items,
        client: { _id: clientId },
        invoiceStripeId: generateId()
      },
      process.env.COOKIE_BUSINESSOWNER.split(','),
    )
      .expect(201)
      .end((err, res) => {
        if (err) return done(err);
        expect(res.body.success).to.be.true;
        invoiceIdTotalZero = res.body.payload.invoiceId;
        done();
      });
  });

  it(`Done A business owner should not be able to checkout with wrong invoice Id`, (done) => {

    invoiceCheckout(
      generateId(),
      process.env.COOKIE_BUSINESSOWNER.split(','),
    )
      .expect(400)
      .end((err, res) => {
        if (err) return done(err);
        expect(res.body.success).false;
        expect(res.body.message).to.eq(invoiceCheckOutGetErrorMessage)
        console.log(res.body.payload)
        done();
      });
  })

  it(`A business owner should not be able to checkout with amount Zero`, (done) => {

    invoiceCheckout(
      invoiceIdTotalZero,
      process.env.COOKIE_BUSINESSOWNER.split(','),
    )
      .expect(400)
      .end((err, res) => {
        if (err) return done(err);
        expect(res.body.success).false;
        expect(res.body.message).to.eq(invoiceCheckOutAmountZeroMessage)
        console.log(res.body.payload)
        console.log(invoiceCheckOutAmountZeroMessage)
        done();
      });
  })
// should be like this Milan said
  it(`A business owner should not be able to checkout with missing payments`, (done) => {

    invoiceCheckout(
      invoiceIdMissingPayment,
      process.env.COOKIE_BUSINESSOWNER.split(','),
    )
      .expect(400)
      .end((err, res) => {
        if (err) return done(err);
        expect(res.body.success).false;
        expect(res.body.message).to.eq(invoiceCheckOutMissingPaymentMessage)
        console.log(res.body.payload)
        console.log(invoiceCheckOutMissingPaymentMessage)
        done();
      });
  })
})
